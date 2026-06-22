# Yggdrasil 落地界限

本文档定义 Yggdrasil 在当前阶段应该落地到什么程度，以及哪些内容必须暂缓。它不是愿景文档，而是给人和 agent 执行任务时使用的边界约束。

Yggdrasil 的核心不是“接一个 LLM”，也不是“做一个 agent 平台”。它当前要训练和验证的是：一个外部 agent 只能提交意图，而世界服务器负责统一裁决、推进状态、记录事件、支持回放。

## 1. 当前阶段目标

当前阶段只做 **单世界、单进程、可持久化、可回放的权威世界裁决内核**。

必须优先落地：

- `Intent`：外部 agent / client 提交的动作意图。
- `World Core`：单一权威裁决路径，负责处理 intent、推进 tick、产生事件。
- `Rule`：最小可组合规则，例如移动、资源消耗、简单交易或任务状态变化。
- `Event`：世界裁决后的事实记录，必须可持久化、可排序、可用于回放。
- `Snapshot`：世界状态快照，先支持粗粒度恢复，不追求复杂增量压缩。
- `Observation`：agent 可见的局部世界视图，不能直接暴露全部权威状态。
- `Store`：MySQL 或可替代的持久化接口，围绕事务边界设计。

可以后置：

- 多世界 / 多区服 / zone 分片。
- 完整 agent runner、记忆、反思、规划、RAG。
- 复杂经济系统、完整任务链、社交关系图。
- gRPC 内部服务拆分。
- RabbitMQ 事件总线。
- Redis 分布式锁、限流和缓存。
- 可视化世界编辑器。

## 2. 一句话边界

如果一个功能不能证明“世界如何权威裁决状态变化”，它就不是当前阶段的主线功能。

## 3. 必须守住的系统原则

### 3.1 世界状态唯一权威

任何会改变世界的写入都必须经过 `World Core`。API、worker、cache、MQ consumer、测试辅助函数都不能绕过世界核心直接修改权威状态。

允许：

- API 把请求转换为 `Intent`。
- worker 处理异步派生任务。
- store 在事务中提交世界核心已经裁决的结果。
- 测试 fixture 初始化世界状态。

不允许：

- HTTP handler 直接改 avatar 位置、背包、任务、资源。
- Redis 缓存被当成最终状态。
- MQ 消费者直接修正权威状态。
- agent 返回的文本被当成已经发生的事实。

### 3.2 Intent 不是 Event

`Intent` 表示“想做什么”，`Event` 表示“世界判定已经发生什么”。任何 PR 只要混淆这两个概念，就应该被要求重做。

典型例子：

- `MoveIntent`：agent 想移动到某位置。
- `MoveRejectedEvent`：世界拒绝移动，原因可能是距离、状态或冲突。
- `AvatarMovedEvent`：世界接受移动，并记录实际起点、终点、tick、版本。

### 3.3 单写多读优先

当前阶段优先采用单进程内的单写路径，不急于拆成微服务。可以并发接收请求，但状态提交必须进入统一裁决队列或统一事务边界。

可接受实现：

- HTTP handler 校验请求后写入 intent 队列。
- tick loop 批量拉取 intent 并排序处理。
- rule engine 只返回裁决结果，不直接提交数据库。
- store 在明确事务中提交状态与事件。

### 3.4 可回放优先于炫技

每次状态变化必须有对应事件。只要事件和快照足够恢复世界，就比引入复杂中间件更重要。

当前验收重点：

- 事件有稳定 ID、world ID、tick、actor、type、payload、created time。
- 同一批 intent 的处理顺序可解释。
- 从空世界或快照 + 事件日志可以恢复到目标状态。
- 拒绝动作也应该可观察或可审计，至少有明确错误结果。

## 4. MVP 功能边界

### 4.1 API

当前只需要最小 REST API：

- `POST /api/v1/worlds`：创建或初始化世界。
- `POST /api/v1/intents`：提交 intent。
- `GET /api/v1/observations/{agentId}`：获取 agent 可见视图。
- `POST /api/v1/admin/ticks`：手动推进 tick，便于测试。
- `GET /api/v1/events`：查询事件日志，便于调试和回放。

暂缓：

- 对外 gRPC。
- WebSocket 实时推送。
- 多租户控制台。
- 复杂 auth / billing / organization。

### 4.2 Domain

最小 domain 集合：

- `World`
- `Agent`
- `Avatar`
- `Position`
- `Resource`
- `Intent`
- `Event`
- `Observation`
- `Rule`
- `Tick`
- `Snapshot`

暂缓：

- 大规模地图系统。
- 完整 NPC AI。
- 复杂装备词条。
- 完整技能树。
- 长链任务编排。

### 4.3 Rule

第一阶段只需要少量规则，但每条规则都必须体现裁决能力：

- 移动规则：校验 avatar 是否存在、是否可行动、目标是否可达。
- 资源规则：校验余额、扣减资源、产出事件。
- 简单交易规则：同一事务内完成双方资源变化。
- 冲突规则：同一 tick 多个 intent 抢同一资源时必须有稳定排序。

不追求：

- 复杂战斗数值。
- LLM 生成规则。
- 动态脚本规则热加载。
- 大规模行为树。

## 5. 数据边界

### 5.1 MySQL / Store

权威数据应该包括：

- world 元信息。
- agent / avatar 档案。
- 当前世界状态。
- event log。
- snapshot 元信息。
- intent 接收记录或幂等记录。

事务边界必须覆盖：

- 状态变更。
- 对应事件写入。
- 必要的版本更新。

不能把外部慢调用放进事务。

### 5.2 Redis

Redis 当前不是必需项。引入 Redis 必须说明它解决的问题，例如：

- 幂等键。
- 限流计数。
- 热点 observation 缓存。
- 短期 session。

不允许把 Redis 作为世界最终真相。

### 5.3 RabbitMQ

RabbitMQ 当前不是必需项。引入 MQ 必须保持世界核心路径仍然可独立正确运行。

适合 MQ 的事情：

- 事件广播。
- 通知。
- 异步索引。
- 回放导出。
- 统计分析。

不适合 MQ 的事情：

- 决定一个 intent 是否成功。
- 替代事务提交。
- 修补世界权威状态。

## 6. Agent 边界

外部 agent 在 Yggdrasil 中是客户端，不是世界本体的一部分。

Yggdrasil 可以提供：

- agent 身份。
- agent 对应 avatar。
- observation。
- intent submission。
- 世界规则反馈。

Yggdrasil 当前不提供：

- LLM provider adapter。
- agent memory。
- agent planning。
- prompt orchestration。
- reflection loop。
- tool-use framework。

如果后续需要这些能力，应该放到独立 `agent-runner` 或独立服务中，通过 API 和 Yggdrasil 交互。

## 7. PR 落地切片

每个 PR 应该只推进一个可验证切片。推荐顺序：

1. 文档基线：项目边界、模块目录、核心概念表。
2. Go 工程骨架：`cmd`、`internal`、基础配置、健康检查。
3. Domain 模型：定义 intent、event、world、avatar、observation。
4. Tick loop：内存版 intent 队列与世界推进。
5. Rule v0：移动规则和拒绝事件。
6. Store v0：事件日志与状态持久化。
7. Snapshot v0：快照保存与恢复。
8. Conflict v0：同 tick 资源抢占排序。
9. Transaction v0：简单交易的事务一致性。
10. Observation v0：按 agent 构建局部视图。

每个 PR 都必须能回答：

- 这次变更守住了哪条世界规则？
- 新增或改变了哪个权威状态路径？
- 对应事件是什么？
- 如何测试裁决、拒绝、回放或事务边界？

## 8. 验收标准

一个阶段性功能只有同时满足下面条件，才算真正落地：

- 有清晰 domain 类型，不靠松散 map 拼业务语义。
- 有明确裁决入口，不绕过 `World Core` 写状态。
- 成功和失败路径都有可解释结果。
- 关键状态变化有事件记录。
- 涉及多个状态变更时有事务边界。
- 至少有单元测试或集成测试覆盖核心规则。
- 文档同步说明该能力属于哪个阶段、解决什么问题、暂不解决什么问题。

## 9. 不做清单

当前阶段明确不做：

- 不做通用 agent 平台。
- 不做 prompt 管理系统。
- 不做 RAG / memory / planning 框架。
- 不做完整 MMORPG 后端。
- 不做微服务优先架构。
- 不做“请求进来直接改库”的 CRUD 世界。
- 不做只有 API 壳、没有裁决和事件的假世界。

这些不是永远不做，而是不能抢占当前主线。

## 10. 文档维护规则

当实现方向变化时，必须同步更新本文档或相关设计文档。尤其是以下情况：

- 引入新的权威写路径。
- 改变 intent / event / snapshot 的含义。
- 引入 Redis、RabbitMQ、gRPC 等基础设施。
- 把某个后置能力提前到当前阶段。
- 新增 agent-runner 或 LLM 相关模块。

如果代码已经改变但边界文档没有更新，PR 应视为未完成。
