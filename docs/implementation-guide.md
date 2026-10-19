# DCP 实现指南

> 配套文档：[规范](specification.md) · [设计论证](design-notes.md)
> 本文给出与语言、框架无关的落地要求，并附 C# 参考契约（其他语言可按同形结构翻译）。参考代码是**契约示意**，不是可编译工程。

---

## 1. 落地目标

按规范 §17 的三个合规档位选择目标：

| 档位 | 内容 | 适用 |
|---|---|---|
| 核心 Core | L0–L3 + 决策日志 + 版本硬门禁 + 结论订阅 | 先跑通日常价值 |
| 标准 Standard | Core + L2 定向咨询 + 责任人登记 + 待答队列 | 需要"单点承诺"可引用 |
| 完整 Full | Standard + L4 会议 + 人类介入深度轴 | 需要多方决议与人机协同 |

实现顺序建议见 [设计论证 §9](design-notes.md#9-落地顺序建议)。

---

## 2. 数据模型

### 2.1 决议记录

决议参与版本语义：同一 `TopicKey` 的 `Version` 单调递增，只有协调者可以写入。

```csharp
namespace Dcp.Contracts;

public sealed record DecisionEntry(
    string Id,
    string TopicKey,                 // 议题键：同一议题的版本序列
    int Version,                     // 从 1 开始单调递增
    string Category,                 // 白名单类别或 Other
    string Content,                  // 决议正文
    IReadOnlyList<string> Participants,
    string? Dissent,                 // 异议记录
    DateTimeOffset At);
```

### 2.2 咨询记录（L1 / L2 / 去重复用）

咨询记录**不参与版本递增、不广播**；其中只有 `Binding=true` 且 `OwnerId` 非空的记录可作为写依据。

```csharp
public enum AdvisoryKind
{
    L1Info,        // 事实澄清，不可引用
    L2Commitment,  // 单点承诺，可引用
    DedupReuse     // 议题去重的复用依据
}

public sealed record AdvisoryEntry
{
    private AdvisoryEntry(
        string id, DateTimeOffset at, AdvisoryKind kind,
        string topicKey, string askerAgentId,
        string question, string answer,
        string? ownerId, bool binding,
        int? basisVersion, string? reclassifiedFrom,
        string? reusedDecisionId, string? reason)
    {
        Id = id;
        At = at;
        Kind = kind;
        TopicKey = topicKey;
        AskerAgentId = askerAgentId;
        Question = question;
        Answer = answer;
        OwnerId = ownerId;
        Binding = binding;
        BasisVersion = basisVersion;
        ReclassifiedFrom = reclassifiedFrom;
        ReusedDecisionId = reusedDecisionId;
        Reason = reason;

        // 唯一事实来源：可引用性由 Binding 与 OwnerId 推导，不接受调用方单独赋值
        Citable = binding && ownerId is not null;
    }

    public string Id { get; }
    public DateTimeOffset At { get; }
    public AdvisoryKind Kind { get; }
    public string TopicKey { get; }
    public string AskerAgentId { get; }
    public string Question { get; }
    public string Answer { get; }
    public string? OwnerId { get; }
    public bool Binding { get; }
    public bool Citable { get; }
    public int? BasisVersion { get; }
    public string? ReclassifiedFrom { get; }
    public string? ReusedDecisionId { get; }
    public string? Reason { get; }

    // 时间与 ID 由协调者在落库时盖章，生产者只填业务字段
    public static AdvisoryEntry Information(
        string id, DateTimeOffset at, string topicKey, string askerId,
        string question, string answer, string? reclassifiedFrom = null) =>
        new(id, at, AdvisoryKind.L1Info, topicKey, askerId, question, answer,
            ownerId: null, binding: false, basisVersion: null,
            reclassifiedFrom, reusedDecisionId: null, reason: null);

    public static AdvisoryEntry Commitment(
        string id, DateTimeOffset at, string topicKey, string askerId,
        string question, string answer, string ownerId, int? basisVersion) =>
        new(id, at, AdvisoryKind.L2Commitment, topicKey, askerId, question, answer,
            RequireOwner(ownerId), binding: true, basisVersion,
            reclassifiedFrom: null, reusedDecisionId: null, reason: null);

    public static AdvisoryEntry DedupReuse(
        string id, DateTimeOffset at, string topicKey, string askerId,
        string question, string answer, string reusedDecisionId, string reason) =>
        new(id, at, AdvisoryKind.DedupReuse, topicKey, askerId, question, answer,
            ownerId: null, binding: false, basisVersion: null,
            reclassifiedFrom: null, RequireText(reusedDecisionId), RequireText(reason));

    private static string RequireOwner(string ownerId)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(ownerId);
        return ownerId;
    }

    private static string RequireText(string value)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        return value;
    }
}
```

> **为什么要工厂**：`L2Commitment` 蕴含"有 owner、可引用"，`DedupReuse` 蕴含"有复用依据"。把构造函数收私有、只暴露工厂，可以让"有承诺却没有 owner"这类组合在类型上无法构造。消费侧据此可以直接读 `Citable`，不需要在每个调用点重复判断。

### 2.3 依据留痕

每次写类操作落一条，记录"提交时依据的是什么、处于什么状态"。它既是审计数据，也是门禁指标的来源。

```csharp
public enum BasisState
{
    Stale,    // 依据的记录已被更新版本取代
    Current,  // 依据的是当前版本
    Absent    // 本次写没有声明依据
}

public sealed record WriteIntent(
    string Id,
    string AgentId,
    string? TaskId,
    string Operation,
    string? BasisRecordId,
    int? BasisVersion,
    BasisState BasisState,
    DateTimeOffset At);
```

注意 `Absent` 的语义是"没有依据机制 / 没有声明"，**不是**"依据过期"。二者不要合并，否则统计会失真。

### 2.4 责任人登记

```csharp
public sealed record OwnershipEntry(
    string WorkerId,
    string Scope,                  // 模块 / 接口 / 文件范围（宿主的表达方式自定）
    DateTimeOffset RegisteredAt,
    int CurrentDecisionVersion,
    DateTimeOffset? RevokedAt);    // 回收后置位，历史保留
```

规则：

- 与派生 Worker **同一时刻**写入，避免出现"Worker 已在工作但无 owner"的窗口；
- 一个责任域同一时刻只允许一个活跃 owner（写分片前提）；
- 查询时只返回 `RevokedAt is null` 的条目；审计查询返回全部。

### 2.5 待答请求

人控模式下，L1 提问、L3 裁决、L4 会议审批、决议审核、危险操作确认都应进入同一个待答队列，并可跨重启恢复。

```csharp
public enum PendingRequestKind
{
    InformationQuestion,   // L1 提问
    Consultation,          // L2 咨询（知会型）
    MeetingApproval,       // L4 开会审批
    DecisionReview,        // 决议审核
    DangerousOperation     // 危险操作确认
}

public sealed record PendingRequest(
    string Id,
    PendingRequestKind Kind,
    string ScopeId,
    string Prompt,
    string RequestedBy,
    DateTimeOffset CreatedAt);
```

入队时盖章 `Id` 与 `CreatedAt`；生产者只填业务字段。

---

## 3. 执行层门禁

### 3.1 调用者上下文

依据必须由**运行时注入**，模型不可写。参考结构：

```csharp
public enum BasisKind { Decision, Advisory }

public sealed record BasisReference(
    BasisKind Kind,
    string RecordId,
    int? Version,            // 决议版本；咨询记录为 null
    bool Binding = false,    // 仅咨询记录使用
    string? OwnerId = null); // 仅咨询记录使用

public sealed record CallerContext(
    string AgentId,
    string? TopicKey,
    BasisReference? Basis);
```

关键点：`CallerContext` 由 Agent 循环在发起工具调用时构造，模型只能提供工具参数，没有任何途径改写 `TopicKey` / `Basis`。

### 3.2 判定结果与接口

```csharp
public sealed record GateVerdict(bool Allowed, string? Reason, string? Diff)
{
    public static readonly GateVerdict Allow = new(true, null, null);

    public static GateVerdict Deny(string reason, string? diff = null) =>
        new(false, reason, diff);
}

public interface IGatekeeper
{
    ValueTask<GateVerdict> CheckAsync(
        WriteOperation operation,
        CallerContext caller,
        CancellationToken ct);
}
```

`GateVerdict` 是**返回值，不是异常**：门禁拒绝是预期内的正常路径（规范 §11.2 要求"拒绝并把差异回灌"）；异常只留给真正的故障。

### 3.3 判定顺序

顺序固定，不可交换：

```
① 只读操作 → 直接放行
② 宿主前置策略（权限 / 路径范围 / 能力边界）→ 越界即拒
③ 依据类型校验
     - 依据是 L1 信息记录 → 拒绝（不可引用）
     - 依据是咨询记录且 (Binding == false 或 OwnerId 为空) → 拒绝
     - 依据是 L2 承诺记录 → 进入 ④
④ 版本校验
     - 当前议题版本 > 依据版本 → 拒绝，并回灌 diff
     - 无依据（Absent）且宿主要求跨域写必须声明依据 → 拒绝
⑤ 危险操作确认 → 需人工确认时走待答队列
⑥ 放行
```

为什么是这个顺序：

| 步骤 | 理由 |
|---|---|
| 前置策略最先 | "任务本就不允许做"，重对齐也不会变合法 |
| 依据类型先于版本 | L1 记录没有版本号；先判类型可以给出更准确的拒绝原因 |
| 版本先于人工确认 | 版本过期重对齐后可能就合法，不该先打扰人 |
| 人工确认最后 | 最贵，避免为注定被拒的操作打扰用户 |

### 3.4 版本比对与回灌

```csharp
int current = log.CurrentVersion(topicKey);
int basis = caller.Basis?.Version ?? 0;   // 从未对齐过该议题按 0 处理

if (basis < current)
    return GateVerdict.Deny("依据版本过期", log.Diff(topicKey, basis, current));

return GateVerdict.Allow;
```

`Diff` 应给出"从依据版本到当前版本之间改了什么"，让 Agent 能重新对齐，而不只是知道"你过期了"。

### 3.5 门禁实例的生命周期

门禁必须是**无状态单例**：共享服务（决策日志、待答队列）通过构造注入，调用者信息与能力配置全部随参数传入。若把"当前调用者"存成实例字段，多 Agent 并发时会串号。

---

## 4. 决策者与转发契约

决策者可为人或 Agent，这是"移交只换角色、不动服务"能成立的接口基础。

```csharp
public enum DecisionOutcomeKind
{
    Unanswered,  // 挂起等待人工答复
    Advisory,    // 只落咨询记录，不递增版本
    Decision     // 落决议记录，参与版本语义
}

public sealed record DecisionRequest(
    string ScopeId,
    string Category,
    string Prompt,
    string RequestedBy);   // 不含时间戳：由落记录时盖章

public sealed record DecisionOutcome
{
    private DecisionOutcome(DecisionOutcomeKind kind, string? content, string? responderId)
    {
        Kind = kind;
        Content = content;
        ResponderId = responderId;
    }

    public DecisionOutcomeKind Kind { get; }
    public string? Content { get; }
    public string? ResponderId { get; }

    public static DecisionOutcome Unanswered { get; } =
        new(DecisionOutcomeKind.Unanswered, null, null);

    public static DecisionOutcome Advisory(string content, string responderId) =>
        new(DecisionOutcomeKind.Advisory, RequireContent(content), responderId);

    public static DecisionOutcome Decision(string content, string responderId) =>
        new(DecisionOutcomeKind.Decision, RequireContent(content), responderId);

    private static string RequireContent(string content)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(content);
        return content;
    }
}

public interface IDecisionMaker
{
    string Id { get; }
    ValueTask<DecisionOutcome> DecideAsync(DecisionRequest request, CancellationToken ct);
}

public interface IDecisionForwarder
{
    ValueTask<DecisionOutcome> ForwardAsync(DecisionRequest request, CancellationToken ct);
}
```

`DecisionOutcome` 刻意不用多态：它是纯数据、无行为差异。用枚举 + 可空字段表达三态，但必须由私有构造函数 + 工厂守住"有答复就必须有内容"的不变量，否则空内容会流进追加写的审计账本。

**人控实现的语义**：在待答队列未落地时，人控决策者应返回 `Unanswered`（挂起），**不要**伪造一个空答复。把"接口已就位"与"作答通道尚未落地"如实分开。

---

## 5. 决策日志与咨询日志

```csharp
public interface IDecisionLog
{
    DecisionEntry? Latest(string topicKey);
    int CurrentVersion(string topicKey);
    IReadOnlyList<DecisionEntry> History(string? topicKey = null);
    string Diff(string topicKey, int fromVersion, int toVersion);

    // 只允许协调者在串行上下文中调用
    void Append(DecisionEntry entry);
}

public interface IAdvisoryLog
{
    void Append(AdvisoryEntry entry);
    IReadOnlyList<AdvisoryEntry> Query(string? topicKey = null);
}
```

**为什么拆成两个接口**：两者共用同一份存储（规范 §11.3 要求同处存放、便于一并审计），但语义相反——决议参与版本递增与广播，咨询记录都不参与。混在一个接口里，调用方会误以为 `AppendAdvisory` 也会影响 `CurrentVersion`。拆开后，"哪些写入影响版本门禁"在类型上直接可见。

**并发要求**：

- 决策日志**单写者**：只有协调者在串行上下文中递增版本；
- 读端（各 Agent 的门禁调用）无锁读取：写入时构造新的不可变索引并整体发布（copy-on-write）；
- 版本号只增不减，不允许跳号后又回退。

---

## 6. 会议协议（L4）

### 6.1 状态机

```csharp
public enum MeetingPhase
{
    Requested,
    PendingApproval,
    Convening,
    InSession,
    Closed
}

public enum MeetingOutcome
{
    Continue,          // 继续推进
    Closed,            // 决议已生效
    ReturnedToLowerLevel  // 退回 L3（议题不合格 / 拒绝 / 超时 / 决议被否决）
}

public sealed record MeetingTransition(
    MeetingPhase? Next,
    MeetingOutcome Outcome,
    string? Reason)
{
    public static MeetingTransition Continue(MeetingPhase next) =>
        new(next, MeetingOutcome.Continue, null);

    public static MeetingTransition Close() =>
        new(null, MeetingOutcome.Closed, null);

    public static MeetingTransition ReturnToL3(string reason) =>
        new(null, MeetingOutcome.ReturnedToLowerLevel, reason);
}

public interface IMeetingProtocol
{
    void Submit(EscalationRequest request);   // 受理一次升级请求
    void Tick();                              // 由调度循环每拍调用一次
}
```

`MeetingTransition` 把"下一个状态"与"这次推进的结果"分开：`Requested`（议题不在白名单）与 `PendingApproval`（拒绝 / 超时）都要**退出会议并降级回 L3**，这不是一个会议状态，用单纯的状态返回值表达不了。

### 6.2 驱动方式

**会议必须由协调者的调度循环逐拍推进，不能一次 `RunAsync` 跑到底。**

- 等待类状态（审批、会合）本拍返回 `null`，调度循环继续做别的事；
- 需要模型参与的状态（并行出方案、仲裁）把工作投给推理循环，结果回填后再由下一拍读取；
- 会议不需要独立线程，也不需要独立同步原语；
- 参会者仍只在安全点响应（规范 §6）。

**审批是"发起一次、之后每拍轮询"，不是 `await`**：

```csharp
// 首次进入时把 Task 与截止时间存到会议会话上
_pendingApproval = approval.RequestAsync(...);
_deadline = clock.GetUtcNow() + timeouts.Approval;

// 之后每拍
if (!_pendingApproval.IsCompletedSuccessfully)
{
    if (clock.GetUtcNow() >= _deadline)
        return MeetingTransition.ReturnToL3("审批超时");
    return null;   // 本拍不推进
}
```

必须用 `IsCompletedSuccessfully` 而不是 `IsCompleted`：后者对故障与取消的 Task 同样为真，会把 `Result` 上的异常抛进调度循环。失败 / 取消 / 超时**一律按拒绝处理**（转 L3），不要让异常掀翻调度循环，也不要让其被当作通过。

**在途任务字段必须随状态迁移清空**：离开 `PendingApproval`、或审核要求重出而退回 `InSession` 时，清空在途 `Task` 与截止时间。否则重出后再次进入时，"首次进入"判不出来，上一轮的已完成 Task 会被误当成本轮结果。

### 6.3 审核必须排在落库与广播之前

为了保住一条不变量：**决策日志的版本递增即"决议已生效"**。

若先落库再审核，被否决的决议会留下一个已经递增的版本号，而它既不广播、也不该被执行；后续所有 Worker 的版本比对基准都被这个"幽灵版本"污染，门禁开始误伤。因此否决按"退回 L3、不产生新版本"处理。

### 6.4 两个开关的读取点

| 开关 | 读取位置 | 行为 |
|---|---|---|
| A：免"批准开会" | `PendingApproval` 入口 | 开 → 直接进 `Convening`；关 → 发起审批并轮询，超时按拒绝 → 退回 L3 |
| B：免"事后审核" | `Closed` 落库之前 | 开 → 直接写日志并广播；关 → 发起审核，通过才落库广播；否决 → 退回 L3；要求重出 → 退回 `InSession`（额外消耗一轮，预算用尽按否决处理） |

### 6.5 去重的落点

去重不归成本护栏，而是在**受理升级请求时**查决策日志：命中相似议题则直接复用上次决议、不开会，并落一条 `DedupReuse` 咨询记录（被复用决议 ID + 判定理由）。这样复用可事后审计。

---

## 7. 并发与不变量

| 不变量 | 做法 |
|---|---|
| 决策日志单写者 | 版本递增只发生在协调者的串行上下文；其他组件只读 |
| 读端无锁 | 写入时构造新索引并整体发布；读端拿到的始终是一份自洽快照 |
| 门禁无调用者状态 | 门禁实例只持有共享服务，调用者上下文逐次传入 |
| 会议单实例 | 同一时刻只允许一场会议；其余请求排队 |
| 队列定序 | 单调递增的接收序号，先到先处理 |
| 消息序号唯一 | 多写者场景用原子自增（C# 用 `Interlocked.Increment(ref _nextSeq)`） |
| 待答请求唯一事实来源 | "请求 ID → TaskCompletionSource" 的并发字典；答复、取消、入队失败三条路径都做一次 `TryRemove`，谁先摘到谁生效 |
| 在途任务随迁移清空 | 见 §6.2 |
| 队列项不随时间增长 | 取消 / 超时 / 失败路径都摘除字典项 |

> 第 7 条的实现要点：不要用"取消标志 + 答复标志"两个字段表达竞争结果，那会产生两个事实来源；用一个原子的 `TryRemove` 决定谁生效。

---

## 8. 存储建议

- 决议、咨询、依据留痕分表存放，咨询与决议可共库不同表；
- 索引：`Decision(TopicKey, Version)`、`Advisory(TopicKey, At)`、`WriteIntent(AgentId, At)`；
- 只追加、不修改（审计账本语义）；作废通过新增记录表达，不删旧记录；
- 版本号由协调者分配，存储层只负责持久化，不做自增；
- 写入失败必须显式上抛，**不允许静默写入空记录**——审计账本声称发生过某事而内容缺失，比直接失败更糟。

---

## 9. 测试矩阵

### 9.1 阶梯与门禁

| # | 场景 | 期望 |
|---|---|---|
| 1 | 引用 L1 记录的写操作 | 拒绝，原因指向"不可引用" |
| 2 | 引用 `Binding=false` 的咨询记录 | 拒绝 |
| 3 | 引用 `OwnerId` 为空的咨询记录 | 拒绝 |
| 4 | 引用 `Binding=true` 且 owner 非空的 L2 记录 | 通过（版本一致时） |
| 5 | 依据版本落后于当前决议 | 拒绝，且回灌 diff |
| 6 | 重新对齐后重试 | 通过 |
| 7 | 只读操作 | 不拦截 |
| 8 | 危险操作且无批准通道 | 拒绝（不得静默挂起或放行） |
| 9 | 判定顺序 | 前置策略先于版本，版本先于人工确认 |

### 9.2 会议

| # | 场景 | 期望 |
|---|---|---|
| 10 | 议题不在白名单 | 退回 L3，不进入审批 |
| 11 | 审批超时 | 退回 L3 |
| 12 | 开关 A 打开 | 跳过审批直接会合 |
| 13 | 参会者超时 | 记为缺席，会议继续 |
| 14 | 开关 B 关闭且审核否决 | 不产生新版本，退回 L3 |
| 15 | 审核要求重出 | 退回 `InSession`；在途任务已清空 |
| 16 | 两场会议并发提交 | 仅一场进行，另一场排队 |
| 17 | 相似议题 | 复用决议并落 `DedupReuse` 记录 |

### 9.3 L2 定向咨询

| # | 场景 | 期望 |
|---|---|---|
| 18 | 责任人正常承诺 | 落 `Binding=true` 记录，可引用 |
| 19 | 责任人答复"只是事实" | 降为 L1，`Binding=false`，不可引用 |
| 20 | 责任人超时 | 提问方回落 L3 |
| 21 | 答复引出约定变更 | 记录 `ReclassifiedFrom=L2`，回流 L3 |
| 22 | 咨询内再发起咨询 | 拒绝（禁止嵌套） |

### 9.4 人控模式

| # | 场景 | 期望 |
|---|---|---|
| 23 | 决策者为人且 L1 / L3 / L4 未答复 | 挂起，不超时回落 |
| 24 | 决策者为人且 L2 责任人超时 | 仍回落 L3 |
| 25 | 应用重启 | 待答请求仍在，可继续答复 |

### 9.5 并发

| # | 场景 | 期望 |
|---|---|---|
| 26 | 多写者同时分配消息序号 | 序号唯一、无重复 |
| 27 | 答复与取消同时到达 | 只有一个生效，另一个返回"已处理" |
| 28 | 门禁并发调用 | 无调用者串号（不同 Agent 的依据不混用） |

---

## 10. 常见实现错误

| 错误 | 后果 |
|---|---|
| 把 L1 记录当写依据 | 事实被当成承诺，认知漂移从后门回来 |
| 咨询记录也递增版本 | 决策日志被咨询污染，广播与版本语义失配 |
| 被否决的决议先落库后审核 | 幽灵版本污染所有 Worker 的比对基准 |
| 门禁拒绝抛异常 | 正常路径被当成故障，调用方错误处理被污染 |
| 门禁实例持有"当前调用者" | 多 Agent 并发时串号 |
| 用 `Task.IsCompleted` 判审批结果 | 故障 / 取消被误判为完成，异常掀翻调度循环 |
| 状态迁移不清空在途任务 | 重出流程读到上一轮结论 |
| L2 找不到 owner 时按 L1 处理 | 承诺被降级为不可引用记录，依赖方无声漂移 |
| 缺席者没有收到决议 | 该 Worker 继续按旧约定工作；应由其下次写操作被门禁拦下兜底 |
| 统计"拦截率" | 同义反复，恒为 1（见设计论证 §5.2） |
| 静默忽略存储失败 | 审计账本出现"声称发生过、内容缺失"的空洞 |

---

## 11. 与规范的对照

| 实现部分 | 规范章节 |
|---|---|
| 五级阶梯、四问判定树 | §2.1 |
| 可引用性与依据类型校验 | §2.2、§11.2 |
| 议题白名单 | §3 |
| 会议状态机 | §4、§9 |
| 人类门禁与待答队列 | §5、§7 |
| 安全点 | §6 |
| 排队与定序 | §8 |
| 结论订阅 | §10 |
| 决策日志与版本门禁 | §11 |
| 成本护栏与去重 | §12 |
| 两个开关、断路器、人控模式 | §13 |
| L2 定向咨询与责任人登记 | §15 |
| 宿主能力要求 | §17 |
