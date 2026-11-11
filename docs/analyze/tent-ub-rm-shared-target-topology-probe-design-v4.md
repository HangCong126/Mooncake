# TENT RM 共享 target、端口拓扑与链路探测恢复专项设计 V4

> 日期：2026-11-11。状态：待实现、可评审的专项设计，不表示当前代码已经具备 RM 共享 target 或主动探测能力。
>
> 代码基线：Mooncake `mainhc / 48aa86c1c27987b221abc4313fb1dad98f2ae519`。母设计：[Jetty 池化与线程池管理 V4](tent-ub-rm-jetty-pool-design-v4.md)。本专项细化其第 4、5、7.4、10、13、14、16、19 节；不改变母设计的单 WR、共享 post 池、完成门和 DMA 安全约束。
>
> 范围：非 bonding 物理 UB、URMA RM、CPU DRAM，一字节单边 WRITE 探测。业务 READ/WRITE 与探测共用对端按口 target，但发送 Jetty 分离。本文不实现业务发送池，不设计 Store 放置算法，不用 probe 成功推断物理端口 GOOD 或业务数据正确。

## 1. 结论与开发门槛

五项核心规则已经确定，不需要重新选择数量模型、触发时机或停止条件：

| 功能 | 本版规则 |
| --- | --- |
| 远端目标 | 每节点每配置端口一个共享 RM target，业务与 probe 共用；endpoint 只持引用 |
| 探测发送资源 | 每个本地端口一个专用 RM 发送 Jetty，不进业务 LIFO 池 |
| 拓扑 | 路径来自合法邻接边；每口两个邻居时，4 口为 8 条有向路径，6 口为 12 条 |
| 探测启动 | 首次真实业务故障立即隔离实际路径，触发或合并该 peer 的有界 campaign |
| 探测停止 | 当前有效成功证据和全部恢复门禁通过后，仅停止该路径任务；其他待核验/故障路径继续 |

可以先开发 Port/Path 模型、邻接校验、PathHealth、campaign、定时与公平调度，并用 fake adapter 验证。实际 probe WR 接入依赖共享 post ticket、typed dispatcher、责任账本与原生资源服务。没有目标 provider 的 RM/屏障/身份依据，不能通过阶段零或启用实机功能。

以下三种状态必须分开记录：

1. **设计已定**：实现可以按本文推进，不再次询问“是否每口一个 Jetty”等已定策略。
2. **W0 接口待联合冻结**：两位开发者需要共用类型、同步协议和契约测试；属于对接前置任务。
3. **provider/实机证据待补齐**：阻塞相应原生能力的正式启用；不阻塞纯软件原型。

详细清单与阻塞范围见第 14 节。

## 2. 当前代码差距与责任边界

当前 UbEndpoint 仍按 RC 创建一组 Jetty、导入并 bind 对端；当前 WorkRequest 没有独立 RM target 引用。当前 RailMonitor 是业务完成统计与 cooldown，尚无本文的 ProbeLinkManager/campaign。不得以“UB 代码已有 Jetty、post/poll”推定已支持共享 target。

`5e399958` 已实现 request_list 级 submitBatch 原子发布，但没有改变逐 slice 单 WR 发送。本文 probe 走独立控制队列，不调用业务 submitBatch，不占 UbTask K；仍复用公共 post/poll、ticket、CompletionLedger 和 typed 事件通道。

### 2.1 两位开发者的唯一主责

A 为 Jetty 池化开发者，B 为 post/poll 与健康开发者。下表是本版建议责任，正式并行联调以前以 W0 共享头文件和测试冻结，不以一侧的文档声明代替双方完成对齐。

| 组件/工作 | A：原生资源与池化侧 | B：post/poll 与健康侧 |
| --- | --- | --- |
| PortInventory、physical-only、RM profile | 主责发现、验证和资源绑定 | 消费稳定 PortId/ContextKey |
| LocalPortTarget、ImportedTargetRegistry | 主责原生创建、导入、撤销、代次与预算 | endpoint/路径/record 持引用 |
| 专用 probe Jetty、源页与探测槽 | 主责创建/注册/发布、退役/删除/补建与 owner | 申请资源使用权，构造 probe job |
| PathAdmissionGate/Lane PostGate/ticket 原语 | 主责共享原语 | 健康隔离时同步关闭路径 gate，post 时认领 |
| PathHealth、fault_epoch、campaign | 读取准入视图 | 唯一权威写入者 |
| probe 定时、peer/port 公平调度 | 提供资源可用性事件 | 主责，经公共 post 池发 WR |
| typed poll/dispatcher、ProbeRecord | 提供原生 PostResult/事件 | 主责消费、结算与 safe 证据 |
| ERROR、flush-done、删除/补建、tombstone | 主责恢复机械 | 提供有限 flush 提取及 onAttemptSafe |
| peer inventory/descriptor 协议 | 主责原生 schema 与授权描述 | 主责引擎装配、缓存更新和健康视图接入，联合评审 |
| 未排空关闭 | 存活 RecoveryService 接收 owner | 移交 record/pin/进展责任，联合验收 |

本文仅引用业务池的只读资源资格，不创建或归还业务成员。健康层不能自行 modify/delete 原生对象；资源层不能依据 probe 结果自行发布路径 HEALTHY。

### 2.2 线程与执行原则

| 执行者 | 工作 |
| --- | --- |
| health/control 执行器 | 有界 timer、campaign、公平调度、握手/导入/门禁核验；不直接 native post |
| 公共 post 池 | 业务单 WR 或受限 probe 单 WR，probe 使用专用发送成员 |
| 公共 poll 池 | 每 JFC 唯一正常消费者；识别真实 WR 后按 namespace 派发 |
| delete/refill 执行器 | A 的统一资源机械，覆盖业务/probe/target 角色 |
| RecoveryService | 关闭移交后持原生依赖并承载继续 poll/ACK/flush 的责任 |

不新增绕过门禁的专用 probe post 线程。控制任务与业务任务分队列、独立额度，post worker 保留控制轮转机会；否则“无业务可恢复”或“业务满载可恢复”无法成立。

## 3. 对象、身份与 owner

### 3.1 原生资源与逻辑对象

| 对象 | 唯一 owner/含义 |
| --- | --- |
| PortBinding | PortId 到经过验证的物理设备、EID/index、ContextKey 的绑定 |
| LocalPortTarget | 本节点一个端口的共享 RM target 及 JFR/JFC、授权等原生依赖 |
| PeerPortTargetDescriptor | 对端发布的按口身份、profile、native target、注册与探测授权信息 |
| ImportedTargetRef | 按本地 Context 和对端 target 代次去重的 import；registry 唯一持 native owner |
| ProbeResource | 本端口专用发送成员、源页/本地 MR、角色 generation 与控制资源 |
| ProbeLink | 一条有向 PathVersion 的目标引用和核验任务视图，不拥有 native 对象 |
| PathHealthRecord | 路径健康、fault_epoch、成功证据、门禁等待责任 |
| PeerProbeCampaign | 一次 peer 核验轮次，快照合法路径、维护逐路径待办 |
| ProbeRecord | 单次 probe 的 token、ticket、WR 责任、target/segment/实际 pin |
| RemoteProbeSlot | 远端节点按 peer/端口分配的专用写入范围、注册与授权；不属于业务 buffer |

endpoint 淘汰只减引用，不直接 unimport 共享句柄；ProbeLink 消失也不能释放尚有 ProbeRecord 使用的对象。native 资源预算、软件引用与实际内存 owner 分层回收。

### 3.2 不可混用的身份

```text
ContextKey = physical_device_identity + EID + context_incarnation
LocalTargetKey = node_incarnation + port_id + target_epoch + rm_profile
ImportedTargetKey = local ContextKey + peer LocalTargetKey + native EID/UASID/ID
PathKey = local ContextKey + peer_incarnation + remote_port_id + rm_profile
PathVersion = PathKey + topology_epoch + target_epoch
LaneKey = ContextKey + native Jetty ID + resource_generation
ProbeKey = PathVersion + fault_epoch + campaign_id + probe_sequence
```

ProbeRecord 另外记录实际使用的 ProbeResource generation、源/远端 Region 代次。若不同物理端口必须区分，ContextKey 应能准确映射对应端口；不能将同一资源的多个配置别名伪装成独立 PathKey。若映射不足以支持要求的路径隔离能力，启动验证明确拒绝该能力。

PortId 不是 eid_index；远端 native ID 不是全局身份；授权 token、MR token 与软件 completion token 采用不同类型/字段。CompletionToken 进程内唯一，零保留给无 WR token 的实体事件，不能复用地址或溢出后绕回。软件 generation 不能自动消除原生 AE/CQE 的 ID ABA。

## 4. 拓扑与数量推导

设本地端口集合 L、peer 远端端口集合 R、经过身份和能力验证的合法邻接 E：

```text
E = {(local_port, remote_port) | 配置允许且双方身份/profile 匹配}
directed_paths(local -> peer) = |E| = sum(degree(local_port))

local shared targets = 本节点已配置并建立的有效端口数
local probe send Jettys = 本节点各有效本地端口的角色目标 1
peer shared targets = peer 按口发布的 target 数
imports = |{(local ContextKey, peer LocalTargetKey) reached by E}|
```

| 场景 | 对端共享 target | 本地 probe 发送角色 | 本方向路径 | 导入关系 |
| --- | --- | --- | --- | --- |
| 双方 4 口，每口两个邻居，Context 各自独立 | 4 | 4 | 8 | 8 |
| 双方 6 口，每口两个邻居，Context 各自独立 | 6 | 6 | 12 | 12 |
| 端口数不等或邻接不规则 | 对端有效口数 | 本地有效口数 | 实际 E 的边数 | 按 Context/TargetKey 去重 |
| 有经验证的 Context 共享 | 仍按对端口发布 | 仍按本地口设角色 | 仍按合法边 | 可能少于路径数；不得据此声称新增独立物理链路 |

8/12 是一个方向的逻辑路径数，不是来回合计，也不证明底层物理路由独立。A 到 B 与 B 到 A 独立建图，不以一方向成功推断反方向成功。

### 4.1 四口邻接示例

| 本地端口 | 允许的远端端口 | 边数 |
| --- | --- | --- |
| P0 | R0、R1 | 2 |
| P1 | R0、R1 | 2 |
| P2 | R2、R3 | 2 |
| P3 | R2、R3 | 2 |

六口相同分组模板可再增加 P4/P5 到 R4/R5，共 12 条边。实现遍历配置列表和邻接，不出现固定四口数组、固定八路循环或定宽四口掩码。

### 4.2 配置草案

以下是结构示例，不是当前受支持配置，device/EID 为占位值：

```json
{
  "topology": {
    "port_count": 4,
    "ports": [
      {"id": "P0", "device": "<device0>", "eid": "<eid0>", "eid_index": 0},
      {"id": "P1", "device": "<device1>", "eid": "<eid1>", "eid_index": 0},
      {"id": "P2", "device": "<device2>", "eid": "<eid2>", "eid_index": 0},
      {"id": "P3", "device": "<device3>", "eid": "<eid3>", "eid_index": 0}
    ],
    "peers": [{
      "id": "<peer-incarnation>",
      "adjacency": {
        "P0": ["R0", "R1"],
        "P1": ["R0", "R1"],
        "P2": ["R2", "R3"],
        "P3": ["R2", "R3"]
      }
    }]
  }
}
```

port_count 只校验列表长度，不据此猜远端端口编号或可达性。自动模板必须先展开成相同的显式邻接，再走同一校验链。首版 topology 在启动时固定；更新采用新 topology_epoch、撤销旧准入并排空旧使用，不能原地改写旧 PathVersion。

### 4.3 启动校验与失败行为

校验 ID 唯一、邻居存在且去重、数量及整数溢出、端口上限、设备/EID/index 真实匹配、physical-only 依据、peer incarnation、target/profile、RM READ/WRITE 能力及探测授权。

对端 inventory 明确列出有效、Unavailable 与配置错误项，不能省略后将“未知”当“健康”。单条边不合法记 INVALID；资源未就绪记 UNAVAILABLE，未取得主动证据的合法路径才是 ELIGIBLE_UNVERIFIED。没有任何可用边时不发布可用传输；是否允许部分端口可用必须在启动结果中显式体现，不能假称完整四口/六口验收通过。

## 5. 按口共享 target 的发布与导入

### 5.1 本地创建与版本握手

1. A 验证 PortBinding 和单一 RM profile，为每端口创建一个 LocalPortTarget，持有其 JFR/JFC/授权依赖。
2. 节点发布完整 inventory、node incarnation、topology_epoch，以及每口 target descriptor；只有依赖就绪、预算已计入、身份已登记的对象可以发布。
3. B/control 按合法邻接准备所需远端 target/探测 Region，A 的 registry 按 ImportedTargetKey 导入。
4. endpoint、业务 attempt、ProbeLink/ProbeRecord 持共享引用；每条 WR 指定对应 imported target，不执行 RC bind。
5. 新 target 生成新 epoch，旧导入留存到其责任结束。新旧描述不就地混用；能力/版本不兼容明确拒绝 RM，不透明切 RC。

建议采用版本化 bootstrap v2 消息；“v2”是目标 schema 名称，尚不是当前 UbBootstrapDesc 已有协议。结构至少包含：

| 描述 | 必须携带并验证 |
| --- | --- |
| 节点/拓扑 | schema/version、node incarnation、端口清单、topology_epoch |
| 端口/target | PortId、Context/设备事实、完整 EID/UASID/native ID、target_epoch、RM profile |
| 能力与授权 | READ/WRITE 能力、访问权限、授权域与 token、双方支持的限制 |
| 远端注册 | segment schema、URMA API/ABI/编码长度、范围与 MR token、注册代次 |
| 探测槽 | 该 peer 的 slot identity、地址/长度、WRITE 权限、owner/注册代次和 target 关联 |

UASID 为零是否合法按 provider 契约判断，不丢弃非零值。所有长度、地址加法、最大消息长度、注册覆盖范围和导入 Context 均校验。裸地址/长度/一个软件 token 不足以构造合法原生 WRITE。

### 5.2 引用与故障域

共享 target 使故障域扩大：同一目标被多个业务 endpoint、probe 和 peer 使用。target 撤销立即关闭所有相关 PathVersion 的新准入，不只影响一个 endpoint。查询压力不随 endpoint 淘汰清零；业务 PathLoadLedger 与 probe 控制负载分账。

本地发送 Jetty 顺序跨 peer 使用不创建新 LocalPortTarget。endpoint/ProbeLink 引用归零不自动触发全局 target 重建，也不重复创建 import。

### 5.3 撤销、重建与入站屏障

| 阶段 | 本地动作 | 远端/引用动作 | 何时允许释放 |
| --- | --- | --- | --- |
| 撤销发布 | 标记旧 target 不可接受新准入，发布版本变化 | peer 停止向旧版本提交新 WR | 尚不允许释放 |
| 收敛使用 | 关闭相关 gate，保留旧对象、授权 MR 和预算 | 旧 imported target/record 继续保活 | 本地出站责任结束不足以证明入站安全 |
| 入站屏障 | 执行 provider 定义的授权撤销/入站 fence | 允许迟到访问按原生契约收敛 | 取得覆盖全部授权访问的安全证据 |
| 发布替代 | 新 native 对象、新 target_epoch、重新验证的 descriptor | peer 按新版本导入，旧结果只结算旧责任 | 旧引用与事件责任结束后回收旧代次 |

peer 停发 ACK、RPC 超时、租约时间到都不能代替入站 DMA 屏障。远端无响应时，必须使用经验证的原生撤销/fence，或继续 quarantine；不能释放仍可能被访问的 target、MR 或实际页。

本地 imported handle 的 unimport 只涉及本地使用责任；共享 target/MR 的入站回收涉及所有被授权 peer。两种操作不能混为同一个“endpoint 删除”。

## 6. 专用 probe 资源与探测内存

### 6.1 每口一个角色，旧责任仍计入

A 为每个本地 PortBinding 维护一个可发布的 probe 发送角色，native 生命周期沿用统一 ERROR/flush/delete/refill/tombstone 机制。B 持只读资源视图与使用权，不拥有 native delete 权。

每端口最多一个 probe 调度槽，状态包括：

```text
FREE -> DISPATCHED -> POSTING -> INFLIGHT -> FINALIZING -> FREE
                         \-> UNKNOWN/DRAINING -> FINALIZING -> FREE
```

DISPATCHED 包含已排入控制 post 队列的任务；POSTING 包含 native post 尚未返回。未知/旧代次在途责任未终结时，该口槽位不释放；提前创建替代 native 对象不意味着可提前发下一次 probe。

健康时资源可常备，周期 probe WR 数为零。新句柄发布需要 generation 和 registry 归属校验，不复用旧 faulted 对象。

### 6.2 源页、远端槽与权限

- A 分配不可变本地源页并在其 Context 注册，record 持本地 Region 和实际 backing owner；payload 只取其中一字节，注册按页对齐/provider 要求。
- 对端节点按 peer/端口分配专用写入槽，持有槽内存和授权注册 owner，经握手发布完整 descriptor。
- 探测槽不得指向业务 buffer；不读回 magic，不将成功 WRITE 解释为远端内容已被业务消费。
- 槽位地址分区不等于原生授权隔离。若同一 MR/token 授权范围覆盖多个槽，必须如实描述权限域；不能宣称只公布子范围就实现 peer 隔离。
- peer 数量、槽总字节数、MR/token/import 数量均有硬上限；超限明确背压/拒绝新 peer，不临时无限分配。

peer 退出后的槽回收服从入站授权撤销；不能因 campaign 结束立即释放对端槽。源页的发送安全与对端槽的入站安全是两个独立域。

### 6.3 JFC 与控制额度

首版使用 dedicated probe JFC shard，由 A 创建、B 唯一 poll owner 消费，数量和深度计入 Context/设备预算。是否一个 Context 对应多个已验证端口不影响每端口 probe 角色数，JFC credit 按真实共享 JFC 汇总。

probe 不占业务 Task K，但必须非阻塞预留端口槽、peer 控制额度和真实 JFC credit。业务不能占尽控制完成容量。marker 不是一条业务 WR 完成，不能按 poll 条数统一扣 credit；它仍占实际 CQ 存储，必须留控制余量。

## 7. 路径资格、健康证据和门禁

### 7.1 三个维度分开

| 维度 | 示例 | 对业务的意义 |
| --- | --- | --- |
| 拓扑资格 | VALID、INVALID、REMOVED | 不合法路径永远不能靠 probe 解封 |
| 资源门禁 | Context/target/profile/Region 就绪，pool/credit 可用 | 有效资源才能准入；池空是容量问题 |
| 路径健康 | ELIGIBLE_UNVERIFIED、HEALTHY、ISOLATED、RECOVERING | 控制是否允许业务尝试 |

Pending/Unknown 是一次核验任务的结果/进展，不是物理端口 BAD，也不应把同口所有 PathHealth 强制覆盖成同一个状态。

### 7.2 状态转换

| 当前状态/事件 | 动作与新状态 |
| --- | --- |
| 启动完成身份/profile/资源验证 | ELIGIBLE_UNVERIFIED；允许受控业务，不启动周期探测 |
| 合格的当前代次业务成功 | 可转 HEALTHY；不能用旧 fault_epoch 的成功覆盖新错误 |
| 首次真实业务错误 | 同步隔离实际 PathVersion、fault_epoch++，关闭业务 path gate，进入 ISOLATED |
| ISOLATED 的当前有效 probe 成功 | 保存证据，进入 RECOVERING |
| RECOVERING 且全部门禁通过 | 原子发布 HEALTHY，停止该路径周期任务，唤醒路径等待桶 |
| 新故障/资源撤销 | 撤销旧证据，关闭准入；按具体故障保留恢复责任或建立新 PathVersion |
| 临时 Context 故障 | 资源阻断，暂停该口 post；资源新代次后重新验证，不能永久停在无出口状态 |
| 永久 REMOVED/Engine stopping | 停止新业务与探测；未终结 record 继续排空/移交，不自动复活 |

错误依据来源必须可追踪：

| 输入 | 路径与资源处置 |
| --- | --- |
| 真实业务错误 CQE/无完成超时 | 发送成员退役；隔离实际路径并触发 peer 核验；超时执行结果 Unknown |
| 本地未 post 的参数/范围验证失败 | 返回请求错误；不凭无 WR 事件推断链路故障 |
| 已接受后的权限/profile 错误 | 保守隔离相应路径/访问资格并退役成员，返回明确原因；probe 成功不能修复错误授权，禁止循环重试原错误请求 |
| 退役诱发的 flush、UNHANDLED、重复 CQE | 只结算旧责任，不递归增加 campaign 或再退役同一成员 |
| pool/K/quota/JFC 不足 | defer，保持健康证据，不记链路失败 |
| probe 真实错误 | 只隔离其明确归属路径；专用发送成员退役，同口其他任务 Pending |
| probe 超时/代次不明/资源未就绪 | 责任或核验状态 Unknown/Pending；不把同口其余路径记 BAD |

首版无需等待 failure_threshold 才隔离，恢复成功阈值为 1 且必须通过完整门禁。告警阈值可以独立配置，不能改变业务隔离规则。

### 7.3 隔离与 post 的原子关系

PathHealth 与 PathAdmissionGate 共享同一权威版本/同步协议。隔离在其线性化点同时更新状态、递增 fault_epoch 并关闭业务准入。claim 与 close 竞争时，要么 post ticket 已合法认领，要么 claim 失败；不能靠异步通知或传一份旧 epoch 快照消除竞争。

probe 的恢复权限与业务权限分开：业务 gate 关闭时可取得受限 probe ticket，但 Context、target 授权、角色生命周期和 stopping 门禁不可绕过。原生 ERROR/删除等待所有已认领 ticket 退出，不在 health 锁内等待 provider。

共享接口要说明这一线性化关系，不要求为了它把健康逻辑搬进资源管理器。

## 8. 错误触发的 peer campaign

### 8.1 启动与合并

1. dispatcher 按真实 token 找到业务 attempt，匹配 LaneKey/PathVersion，并一次性识别首次故障。
2. 同步关闭该资源/路径准入，记录 fault_epoch 和错误原因，再将资源退役工作交 A 的恢复机械。
3. 为该 peer 创建或合并唯一活动 campaign，快照当前合法 PathVersion；未故障路径保留原业务资格。
4. 为快照中每条路径登记一次首轮核验需求；每路径仅保留一个有界待办，不按每条错误创建 timer。
5. 新错误到来时重新激活需要核验的路径，去重更新需求代次；实际出错路径的新 fault_epoch 必须使旧成功失效。多次相同唤醒合并，不能饿死尚未获得首轮机会的路径。

campaign 本身不持原生 Jetty；ProbeRecord 才持一次 WR 的具体资源引用。CampaignId 在当前 peer incarnation 内唯一，结束后新一轮使用新 ID。

### 8.2 待办状态

| 任务状态 | 含义 |
| --- | --- |
| NEED_VERIFY | 首轮或新故障要求核验 |
| WAIT_RESOURCE | 等 Context/target/probe 角色/控制额度，不计失败 |
| DISPATCHED/INFLIGHT | 控制队列或 WR 已占用端口调度槽 |
| RETRY_AT | 明确失败路径按退避等待下一次核验 |
| WAIT_GATE | 已有合格成功证据，等待外层门禁/控制核验 |
| VERIFIED | 本轮有效成功且无需后续恢复任务 |
| OBSOLETE | PathVersion 被替换；仅旧 record 继续结算 |

这些状态与 PathHealth 分开。正常路径首轮 probe 未发出时只是 NEED_VERIFY/WAIT_RESOURCE，不应提前变为 ISOLATED。

### 8.3 公平性、周期与容量

每端口对 peer 轮转，再对该 peer 的到期路径轮转；使用稳定游标，记录下一次到期时间。一个端口已有调度槽占用时不能调度第二个 probe。不同端口可以有在途 WR 并行，公共 post worker 不阻塞等待 probe 完成。

每 peer 控制上限和全局任务/定时器上限有界；首轮需求不被同一故障路径的连续重试覆盖。以 registry 待办和最小堆/时间轮调度；有界唤醒队列满时不删除首轮需求。资源可用性变化唤醒 WAIT_RESOURCE，timer 负责到期重试，stopping 无条件唤醒清理。

明确失败采用指数退避＋抖动，有上限；无资源是等待，不累计失败次数。probe 超时先排空旧责任，不能只等下一个 interval 就再发。计时从 native post 认领/执行边界记录，不把控制队列等待算作网络 RTT；另记排队时延。

### 8.4 campaign 结束

只有本轮快照的首轮需求全部收敛，且没有 RETRY_AT/WAIT_RESOURCE/WAIT_GATE/在途责任需要推进时，才结束 campaign。拓扑被替换的项明确标 OBSOLETE，不能当成功；peer 被移除时停止其任务但保留旧 record 到安全结束。

某条路径恢复时只停止它的周期任务，不取消其他路径的待办。campaign 退出不销毁共享 target、probe 角色或对端槽，只停止本轮调度。

## 9. 单次 probe 的 post、完成与退役

### 9.1 WR 与资源责任

```text
Operation = WRITE
payload length = 1
complete_enable = 1
next = null
target = 当前 Context 导入的该 peer/远端口共享 target
remote address = 握手授权的专用探测槽
completion token = 唯一 probe namespace token
```

ProbeRecord 至少保存 ProbeKey、LaneKey、resource_generation、source/remote Region 引用、target 引用、实际 source pin、端口槽/控制/JFC reservation、PostTicket、post_returned、acceptance、wr_terminal、handler_refs、fault_seen、finalize_claimed。

这些是目标类型。当前 WorkRequest/Completion 不能直接满足 RM target、typed marker 和三态接受契约，需与 A 联合扩展。

### 9.2 投递流程

1. health 调度器只将当前到期、资源可能就绪的 ProbeJob 放入受限控制队列，并以端口槽认领防止重复调度；不执行 provider/RPC。
2. post worker 复核 job 的 PathVersion/ProbeKey、stopping 和资源视图。缺 target/Region 交控制面准备，释放未执行使用权并等待通知。
3. 非阻塞预留专用控制/JFC credit，取得专用 probe 资源使用权，挂全部 owner，安装 token/record 与原生责任引用。
4. 在共享协议中认领 probe PostTicket；复核 Context、角色 generation、target/Region 代次和 probe 权限。不仅是“读 generation 然后直接 post”。
5. 调用一次 native post。post 返回与 CQE 都推进同一 ProbeRecord 完成协议；post 调用栈计入访问责任。
6. 确定未调用 native post 的失败逆序回滚并重排。NotAccepted 按明确原因决定退避或请求控制修复；Accepted/Unknown 不重复发当前 attempt。

| 接受结果 | probe 处理 |
| --- | --- |
| NotAccepted 且确定无 WR | post/handler 退出后释放资源责任；纯容量/关闭竞争等待，真实提交错误保留原因，按资源故障契约处置 |
| Accepted | 保留 record，等待真实 CQE；成功也须等 post 返回再释放端口槽 |
| Unknown | 关闭该 probe 发送资源、交统一排空；保留 pin/credit/槽，不重发 |
| 真实 CQE 与 NotAccepted 冲突 | 按 provider 契约冲突隔离，保留诊断与尚未证明安全的责任，不覆写事实后假装正常 |

NotAccepted 表示执行责任确定未产生，不代表所有错误都应该重试，也不代表可忽略 provider 错误。accepted 状态映射由阶段零冻结。

### 9.3 完成分类与 safe finalize

adapter/dispatcher 先按类型及 flag/status 识别实体 marker，只有真实 WR 才读 user_ctx；然后将 probe token 交本模块。AE 使用独立获取/ACK 通道。marker、退役 flush、WR_UNHANDLED 按统一 dispatcher 结算，不能当业务成功。

probe safe finalize 必须同时具备：

```text
post 已返回或确定不调用
AND 本次 WR 责任已终结或有可信排空证据
AND 所有相关 handler/native 访问已退出
AND 一次性 finalize 认领成功
```

健康证据发布另受第 10 节约束。safe finalize 释放端口调度槽、控制/JFC credit、source pin 和相关引用；native 对象是否删除由 A 决定。stale 记录仍要正常归还责任，不能因“结果无效”直接丢弃。

### 9.4 真实错误、超时与专用 Jetty 替换

真实错误对应的 probe 发送对象不复用；A 执行 ERROR、send flush-done、软件 flush、DELETE_READY、删除和补建。B 暂停该口新投递，同口其他待办保留 Pending；新 generation 发布后，仍须等旧端口槽安全释放才能轮转。

timeout 表示当前 probe 结果与 DMA 责任不明：保留源页、target/Region、credit 和端口槽，登记恢复责任。没有 CQE、资源不足、旧对象迟到事件不能被汇总成整个端口物理故障。

flushDrained 仅证明软件提取取空，不证明 post/native/handler 全退出；A 另须取得共享责任账本的 onAttemptSafe 证据。资源恢复队列满时，以权威 registry 中的待办状态保留责任，不在 poller 中阻塞或丢事件。

## 10. 恢复证据、停止任务与新故障竞争

### 10.1 合格成功证据

一次 probe 成功用于恢复必须满足：

```text
真实 SUCCESS WR，接受与完成事实无契约冲突
AND ProbeKey 对应当前 PathVersion
AND peer/context/target/Region/probe-resource incarnation 与代次有效
AND 提交时 fault_epoch == 当前 fault_epoch
AND 没有更晚业务/资源故障
AND 节点/端口业务门禁允许
AND 业务 READ/WRITE profile 与权限有效
```

成功 WRITE 仅证明这个 target/探测槽配置的当前可用性，不证明业务 READ 权限、大长度、物理端口 GOOD、其他方向或用户数据内容正确。证据在身份/故障/授权变化时撤销；不以单纯“经过了一段时间”自动恢复资格。

合格的业务成功可以将 ELIGIBLE_UNVERIFIED 转为 HEALTHY。已 ISOLATED 的恢复按 probe/控制证据流程推进，迟到旧业务成功不直接解封。

### 10.2 原子发布与逐路径停止

在与故障事件相同的同步协议内：

1. 校验 ProbeKey 和全部门禁。
2. 若通过，发布 HEALTHY/允许业务，并递增该路径任务代次以作废未 post 的旧 job/timer。
3. 清除该路径周期待办，保留已 post ProbeRecord 到 safe finalize。
4. 更新路径 availability epoch，唤醒相关业务 defer。
5. 新故障若随后获胜，更新 fault_epoch、重新隔离并建立新需求；旧 cancel 不能删除新任务。

取消 timer 只是停止未来调度，不能证明旧 WR 已结束；任务代次只防逻辑旧任务，不替代原生事件屏障。

### 10.3 RECOVERING 的持续责任

probe 成功但外层 gate 不允许时保持 RECOVERING/WAIT_GATE，登记有界控制核验。已有成功证据仍有效时，无需持续发送周期 WRITE；由资源/端口证据事件及受限门禁重查推进。若期间发生新故障或代次变化，撤销旧证据并重新核验。

peer/资源事件丢失时，权威 registry 的待办扫描保证进展；控制核验超时应报告仍被阻断及原因，不能静默取消最后一个任务导致永久停滞。

### 10.4 物理端口与节点资格

路径 ISOLATED 是保守业务策略，不等于物理端口 BAD。只有完整非空端口集、准确归属、当前实例有效 BAD 证据才能形成节点全坏锁存。pool 空、ACK timeout、Pending、证据过期、查询失败不构成全坏，也不自动解封。

只有具备物理事实契约的合格新 GOOD 才能解除节点锁存。没有该契约时，probe 成功只恢复相应路径证据；若物理门禁仍关闭则继续 WAIT_GATE，由控制面或资源重建提供证据。

## 11. 生命周期、拓扑更新与关闭

### 11.1 启动

验证端口/邻接与 provider profile；创建共享 target、源页/授权槽与 probe 角色；建立有界 registry 与引用；准备合法边的 imported target/Region；分配 JFC 唯一 poll owner；初始化 PathHealth；启用公共 post 的控制轮转。

ELIGIBLE_UNVERIFIED 可以业务尝试，不启动常驻周期 WR。初始化缺失资源由控制面补齐，不令 post worker 首次创建 Jetty 或持使用权等待 RPC。

### 11.2 target/peer/Context 更新

| 变化 | 处理 |
| --- | --- |
| target_epoch 更新 | 关闭旧路径 gate，旧 record 保活；新描述重新导入和校验，建立新 PathVersion |
| peer 重启 | 新 peer incarnation；旧 campaign/证据不得迁移为健康，旧责任另行收敛 |
| topology_epoch 更新 | 新图完整校验并发布，旧版本撤销准入；不原地修改旧键 |
| probe 资源重建 | 更新 role generation、按口发布资源事件；不清除旧记录/端口槽的责任 |
| Context 重建 | 新 ContextKey，重新注册/导入，等待新资源就绪后重建资格；不得复用旧 CQ/实体身份 |

### 11.3 关闭与未排空移交

停止新 campaign/新 job 和新 post 认领，取消未提交任务并退还调度槽/控制预留；等待公共 post 调用栈退出。poll/ACK/flush 在原生责任结束前继续运行，回调通知按 V4 的安全完成与 outbox 协议排空。

关闭返回 Draining/Unproven 时，使用强 owner bundle 移交 ProbeRecord、source pin、target/Region、Context/JFC/runtime lease、未结 credit/预算、事件与通知责任。接收方登记并确认后，原方才能撤销自身责任。继续消费事件的执行器必须明确，不能把 registry 移空后直接停止最后 poller。

共享 target 和对端槽的入站屏障由 A 提供；本地 probe 发送排空不能替代它。进程终止后的页回收和远端访问撤销属于目标内核/驱动契约，用户态 owner 不可能跨进程继续存活。

## 12. 共享接口与模块落位

### 12.1 W0 接口语义草案

以下不是当前已存在 API；名称可调整，身份、owner 和同步语义不能分叉：

```text
snapshotPortInventory() -> PortInventorySnapshot
publishLocalTargets(inventory_version) -> TargetInventory
preparePeerTargets(peer_descriptor, adjacency) -> Ready | Pending | Invalid

snapshotProbeResources(port_id) -> ProbeResourceView
tryAcquireProbeUse(port_id, probe_key) -> ProbeUse | Unavailable(key, epoch)
tryClaimPost(probe_use, probe_admission_gate) -> PostTicket | GateClosed
attachAttempt(probe_record, probe_use) -> ResponsibilityRef
onPostReturn(ticket, PostResult)
onWrTerminal(token, TypedEvent)
onAttemptSafe(responsibility_ref, evidence)

isolatePath(path_version, fault_cause) -> new_fault_epoch
triggerOrMergeCampaign(peer, path_version, fault_epoch)
requestProbe(job)
tryRecover(probe_key, success_evidence) -> Healthy | WaitGate | Stale

beginErrorOnce(lane_key, cause)
onAvailabilityChanged(resource_key, epoch, reason)
handoffRecovery(owner_bundle) -> HandoffReceipt
```

isolatePath 和 post claim 共用门禁协议，beginErrorOnce 先同步关闭原生使用资格再异步 ERROR。仅定义同名结构、传递 fault_epoch 快照或异步 enqueue 不构成同步证明。

### 12.2 必须包含的资源通知

| 通知 | 本模块动作 |
| --- | --- |
| target/Region 准备完成 | 更新权威版本，唤醒相应 WAIT_RESOURCE |
| 控制/JFC credit、端口槽释放 | 唤醒受同一资源限制的公平调度 |
| probe replacement ready | 更新 generation，复核旧槽责任后轮转 |
| target/Context 撤销 | 立即阻断相应 post，保存旧 record/证据归属 |
| 新 Context/target/topology ready | 建立新版本引用、重新验证，更新等待项 |
| 门禁事实变化 | 推进 RECOVERING，不要求业务先成功 |
| stopping/cancel | 无条件清理未 post job，不等待无关资源事件 |

所有队列有界、合并，权威 registry 留待办，丢一个唤醒不能丢任务或屏障。resource key 包括真正限制等待的 port/Context/target/JFC/peer，而不是只有“pool 不足”一个全局桶。

### 12.3 建议文件划分

| 模块 | 主要职责/负责人 |
| --- | --- |
| port_inventory、rm_target_registry、probe_resource_provider | A：物理绑定、target/import、角色与探测内存 |
| path_health、probe_campaign、probe_scheduler | B：资格、轮次、计时、公平和任务停止 |
| post_scheduler、completion_ledger、typed_dispatcher | B：公共数据面，probe 复用同一套门禁与安全结算 |
| recovery_registry、resource_refill、target_revoke | A：资源故障、角色补建、入站回收 |
| bootstrap schema、shared gate/types | A/B 共用定义与契约测试 |
| UbEngine/UbTransport 装配 | B 主责接入，A 提供服务，联合关闭 |

先定义 ub_core 的窄接口与独立身份类型，ub_health 不反向依赖 Store；不在旧 endpoint 中再复制一套探测 Jetty/target owner。

## 13. 配置与容量

以下为目标配置语义；当前 params.h 尚不支持。示例数值是开发实验起点，不能标为已验收默认值。

| 配置 | 规则/建议 |
| --- | --- |
| topology.ports/peers/adjacency/template | 按第 4 节解析，首版启动时固定 |
| rm_profile/physical_only | 明确 RM profile；未知身份不猜测 |
| probe.enabled | 启用本专项功能；生产 gate 还须通过第 14 节 |
| probe.interval_ms | 建议实验值 1000，故障路径重试基本周期 |
| probe.backoff_max_ms | 建议实验值 30000，含抖动，算法受总停止/版本条件约束 |
| probe.timeout_ms | 建议实验值 1000，无完成表示责任不明，不直接重发 |
| probe.peer_budget | 建议实验值 4；不改变每端口单调度槽 |
| probe.stop_on_recovery | 固定 true |
| probe.recover_success_threshold | 固定 1，附加全部身份/门禁校验 |
| probe.port_inflight | 固定 1，涵盖 DISPATCHED、post 未返回和旧代次 Unknown |
| probe.jfc_mode | 首版 dedicated，shared 模式不作为首版验收前提 |
| probe.jfc_depth/控制队列/待办/peer/slot 上限 | W0 给出有限配置，阶段零与性能实验验证数值 |
| probe.failure_alert_threshold | 可选，仅告警；不延迟首次隔离，不自动放开健康门禁 |

总预算同时覆盖 active、creating、retiring、quarantine 的业务/probe/target 及全部依赖。四口/六口仅能推导 active probe 角色 4/6，不能据此推导设备总资源够用。对端 target 不随 peer 数增加，但 slot/MR/import 与控制任务会增加，必须分别限制。

每口 source pin、每 peer 远端槽、旧代次留存和替换余量单独记账。容量满是背压或 WAIT_RESOURCE，不是链路 BAD。

## 14. 待确定点是否阻塞开发

### 14.1 当前判定

没有需要重新决定的五项业务策略。尚未完成的工作分为以下范围，不能一概宣布“全部无法开工”：

| ID | 待确定/验证内容 | 谁完成 | 阻塞什么 | 未完成时仍可做什么 |
| --- | --- | --- | --- | --- |
| C1 | 共享身份/descriptor/API、typed event、probe ticket、fault_epoch/gate 线性化、责任回执与交接 | A/B，W0 | 两侧真实接口联调与稳定实现定稿 | 按本文语义写共用草案、fake 和契约测试 |
| P1 | 物理 RM、多 peer 共用 target、逐 WR target、三态接受、错误归属、AE/flush/delete/ID 重用 | A 主责，B 联合验证 | 原生 adapter 契约正式启用、实际 probe 和阶段零通过 | 图/健康/campaign、fake 交错与软件调度 |
| P2 | shared target/MR/探测槽的授权撤销与入站 DMA fence | A 主责，目标 provider 证据 | 安全 target 重建、槽/MR/Context 回收和完整 shutdown 验收 | 保留旧资源的原型、版本协议与入站恢复状态机；不宣称安全销毁 |
| P3 | physical-only 及 PortId 到真实 device/EID/Context 的可信对应 | A/目标环境 | 实际 4/6 口映射、物理故障域声明与启用 | 使用虚拟明确身份跑逻辑拓扑测试，不宣称物理端口验收 |
| E1 | 目标设备/对端/URMA/UDMA/驱动/profile、权限和故障注入环境 | 两侧共同 | 实机验收与发布 | 软件测试、资源边界模拟和构建；不把未核实环境记为已不存在 |
| T1 | 时延/吞吐/CPU、控制额度、peer/slot 数与 JFC 深度具体阈值 | A/B，目标负载实验 | 性能验收和生产配置冻结 | 使用有限实验配置验证功能；不预先保证恢复上界 |

P1/P2/P3 是原有阶段零外部契约，并非本文新引入的业务设计悬案。C1 是开工初始共同任务，不能由 B 在缺乏 A 的 ticket/owner 语义时直接持裸句柄 post。

### 14.2 不应再次提出为全局阻塞的问题

| 问题 | 已确定答案 |
| --- | --- |
| 每 endpoint 要不要各建 target | 不建，按节点端口共享；endpoint 持引用 |
| probe 要不要进入业务池 | 不进入，专用每口角色 |
| 是否初始化即永久周期探测 | 否；资源常备，错误触发任务 |
| 一个错误是否把 peer 全路径置坏 | 否；只隔离实际路径，全集合受限核验 |
| probe 成功是否复活旧发送 Jetty | 否；路径健康与 native 对象生命周期分离 |
| 恢复是否还常驻发包 | 不发该路径的周期 probe；其他任务继续 |
| Unknown probe 是否减计数后重发 | 不允许；先收敛旧责任 |
| probe WRITE 成功能否证明物理 GOOD/业务 READ | 不能，必须另有事实/profile 契约 |
| pool 空/JFC 满是否熔断路径 | 不熔断，等待资源 |
| 4/6 口是否固定数组 | 否，按合法邻接动态计算 |

### 14.3 W0 需要产出的可评审结果

1. A/B 共享的 Port/Target/Path/ProbeKey、descriptor 与 typed event 定义。
2. probe use/ticket、同步隔离、onPostReturn/onAttemptSafe 的契约及 mock。
3. target 撤销/入站屏障的 provider 证据清单，缺证据时的 quarantine 路径。
4. 有界资源通知、旧代次替换和强 owner 移交协议。
5. 一套共同验收用例与有限配置，双方确认相同失败分类。

这些结果冻结后可以并行实现资源服务和健康调度；当前文档的生成不自动表示两位开发者已经完成接口确认。

## 15. 实施顺序与交付

| 阶段 | A 工作 | B 工作 | 进入联合下一阶段的条件 |
| --- | --- | --- | --- |
| S0/W0 | 身份/descriptor、ticket、typed event、原生能力证据表 | PathHealth/事件模型、契约测试、bounded 待办 | C1 类型与同步规则一致 |
| S1 逻辑拓扑 | Port/peer inventory fake 与 target 视图 | 动态邻接、健康状态与唯一 campaign | 4/6/不对称/无效映射软件测试 |
| S2 原生资源 | RM target/import、源页/远端槽、专用 probe 角色 | 公共 post/poll、ProbeRecord、专用控制队列 | P1/P3 阶段零证据；软件 early-CQE/Unknown 通过 |
| S3 故障与恢复 | ERROR/flush/delete/refill、入站撤销、旧代次账本 | 公平轮转、退避、RECOVERING、逐路径停止 | P2 与责任回执明确；真实故障不提前释放 |
| S4 上层与关闭 | 存活 RecoveryService、预算和资源销毁证据 | gate/availability 接入、owner 移交和通知 | 无业务恢复、满载恢复、关闭交错与实机验收 |

S1 可以先做，不需要先拿到硬件。S2 的真实投递不能跳过 ticket/record/dispatcher，S3 的资源销毁不能跳过入站屏障。构建成功或 fake 通过不等于上述 provider 条件已通过。

## 16. 验收矩阵与观测

| 场景 | 必须证明 |
| --- | --- |
| 四口/六口/不等端口数/不规则邻接 | 路径分别按 E 推导，4/8/4 与 6/12/6 资源含义正确，不猜线缆 |
| 重复端口、错误 EID、Context 别名 | 明确拒绝无依据映射，不伪造物理隔离能力 |
| 多 endpoint/多 peer/业务与 probe 共用 target | native target 不随 endpoint 增长，导入去重，WR 指向正确 target |
| endpoint 淘汰、peer 退出 | 不 unimport 其他活跃引用；槽回收不绕过入站屏障 |
| 启动后没有业务错误 | 可保留资源，但没有周期 probe WR |
| 单业务错误 | 实际路径立即隔离，peer 其余路径保持资格并各获首轮机会 |
| 相同/并发错误 | 每 peer 一个 campaign，每路径一个待办，fault_epoch 去重规则正确 |
| 多 peer 多路径共用本口 probe | 同口最多一个调度槽含 Unknown，公平轮转，其他口可并行 |
| 业务满载、控制队列满、资源不足 | probe 有进展机会；待办保留；Pending 不记 BAD |
| marker 的 user_ctx 为零或随机 probe 值 | 先识别实体 marker，不误派到 PLM，不丢 flush-done |
| 真实 probe CQE 带 flush/suspend flag | 保留有效 token，结算旧 WR，不误作实体 marker |
| CQE 早于 post 返回 | 不早释放端口槽、源页、credit，不重复 finalize |
| NotAccepted/Unknown/状态冲突 | 分别安全回滚、保留排空、隔离诊断；不按未知结果直接重发 |
| probe Jetty 错误/超时及重建 | 错误对象退役；其他路径 Pending；新代次不能越过旧责任 |
| 一条路径恢复，其他故障仍在 | 只停恢复路径周期任务，campaign 继续推进其他项 |
| 恢复后没有业务 | 无需业务再次驱动恢复，且已恢复路径不周期发包 |
| 旧成功与新故障竞争 | 完整 ProbeKey 和共享 gate 阻止错误解封/旧 timer 取消新任务 |
| probe 成功但 node/profile gate 未通过 | 保持 RECOVERING，控制核验继续；无虚假业务资格 |
| target/peer/topology/Context 更新 | 建立新版本，旧记录只结算旧责任，不复用旧证据 |
| target 无响应 peer 的撤销 | 无入站证据不释放目标/MR/实际内存，不用超时冒充 fence |
| physical GOOD/BAD 与 Pending 混合 | 不误形成全端口坏或自动解封，不将 WRITE 成功当 READ 授权 |
| stop 与 post/refill/poll 交错 | 停新任务，保留并移交未收敛 owner，最后消费者不提前退出 |

观测至少包括每口资源 generation/调度槽、active probe/retiring/quarantine/源页 pin、peer slot/MR/import 数；campaign 数和逐路径待办/首轮覆盖；fault_epoch、成功/失败/Unknown/stale、退避与停止次数；控制等待/JFC credit/调度公平；恢复时延、post/poll CPU、poll 暂停和 owner 移交。

性能验收按相同预算/版本/负载记录。首轮覆盖时延取决于路径数、peer 数、同口单在途、WR 时延及退避，不能无条件承诺一个 interval 内完成全部核验。

## 17. 文档关系、源码与验证边界

- [母设计 V4](tent-ub-rm-jetty-pool-design-v4.md)：业务池、完整完成门、SubmissionTxn、两人总分工、预算和安全接入；本专项不修改这些已定规则。
- [ub_transport.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/ub_transport.cpp)：当前 endpoint bootstrap 与 submitBatch 接入，尚不是按口共享 target 协议。
- [endpoint.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/endpoint.h)、[endpoint.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/endpoint.cpp)：当前 RC Jetty 所有权，需要由共享 target/路径引用替换。
- [topology_attrs.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/topology_attrs.h)、[device_selection.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/device_selection.h)：当前 EID/设备元数据和 bonding 名称启发式，不能直接证明物理口身份。
- [urma_adapter.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/urma_adapter.h)、[urma_adapter.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/urma_adapter.cpp)：RM target、三态 post、typed event 与 probe 原生契约的改造边界。
- [workers.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/workers.cpp)、[rail_monitor.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/rail_monitor.cpp)：当前 post/poll 与被动 cooldown，健康资格需接入本专项状态机。
- [V3 固定来源](https://github.com/HangCong126/Supercache/blob/29db96ffceed1493fa81ec5ab2d8b426cd995449/docs/analyze/tent-ub-rm-jetty-pool-design.md)：本专项五条规则的来源。UMDK 参考 commit 为 09086da5d86c53f83eb395109d636957d3efcc79；源码依据不能冒充目标环境实机结论。

本次仅新增专项 Markdown，不更改 V4、代码、开关或站点导航。第 16 节是后续实现验收要求，不表示已执行 C++/真实 UB 测试。交付时检查 UTF-8/围栏/表格/JSON/本地链接与空白；Sphinx/pre-commit 的实际可运行情况单独说明，不把工具缺失记为通过。
