# Yggdrasil 实施进度

本文档记录 Yggdrasil 从文档基线走向可运行世界内核的实施顺序、当前状态和每个阶段的完成定义。

当前判断：项目还没有代码实现，最重要的下一步不是补齐中间件，而是先做出一个最小但真实的 **权威世界裁决闭环**。

## 1. 当前状态

已完成：

- 项目定位已明确：Yggdrasil 是多 agent 接入型权威世界模拟服务器，不是通用 agent 平台。
- 设计主线已明确：外部 agent 只能提交 `Intent`，世界核心裁决后产生 `Event`。
- 落地界限已明确：当前阶段只做单世界、单进程、可持久化、可回放的世界内核。
- PR 协作规则已明确：每个 PR 使用 `Summary / Why / Implementation / Test / Notes` 结构。

尚未开始：

- Go 工程骨架。
- domain 类型。
- world core / tick loop。
- rule engine。
- event log。
- replay。
- REST API。
- store transaction。

## 2. 当前阶段目标

第一阶段目标是完成一个可以被测试驱动的内存版世界内核：

```text
create world
  -> register agent/avatar
  -> submit intent
  -> advance tick
  -> adjudicate by rule
  -> emit accepted/rejected event
  -> update authority state
  -> build observation
```

这个闭环必须证明：

- 请求只是意图，不是事实。
- 状态变化只能由 `World Core` 裁决产生。
- 成功和失败路径都有明确事件或结果。
- 同一批 intent 的处理顺序稳定。
- 事件日志具备后续 replay 的基础。

## 3. 推荐 PR 顺序

### PR 1: 文档基线

目标：

- 合入落地界限、实施进度、agent 协作 PR 规则和 PR 模板。

完成定义：

- README 有清晰文档入口。
- `.github/pull_request_template.md` 存在。
- 后续 agent 可以根据文档判断做什么、不做什么、如何写 PR。

测试：

- documentation-only change。
- 手动确认 README 链接目标存在。

### PR 2: Go 工程骨架

目标：

- 初始化最小 Go module。
- 建立目录骨架，但不写复杂业务。

建议目录：

```text
cmd/yggdrasil
internal/domain
internal/world
internal/rules
internal/store
internal/api
```

完成定义：

- `go test ./...` 能运行。
- `go run ./cmd/yggdrasil` 至少能启动并输出基础版本或健康信息。
- 没有引入 MySQL、Redis、RabbitMQ、gRPC。

### PR 3: Domain v0

目标：

- 定义最小核心类型。

建议类型：

- `WorldID`
- `AgentID`
- `AvatarID`
- `Tick`
- `Position`
- `Intent`
- `Event`
- `Observation`
- `WorldState`

完成定义：

- `Intent` 和 `Event` 语义分离。
- 类型命名能表达世界规则，不靠散乱 `map[string]any` 承载核心业务。
- 有基础单元测试覆盖 ID、position、event 构造或校验。

### PR 4: In-Memory World Core

目标：

- 实现内存版世界核心和手动 tick。

完成定义：

- 可以向 world 提交 intent。
- tick 时统一处理队列。
- world state 写入只有一个入口。
- tick result 返回本 tick 的 accepted / rejected 结果。

测试重点：

- 空队列 tick 不改变状态。
- 多个 intent 处理顺序稳定。
- 无效 actor 不会改状态。

### PR 5: Movement Rule v0

目标：

- 实现第一条可裁决规则：移动。

完成定义：

- `MoveIntent` 不直接改变位置。
- tick 裁决成功后产生 `AvatarMovedEvent`。
- tick 裁决失败后产生 rejected result 或 `MoveRejectedEvent`。
- avatar 不存在、目标不可达、状态不可行动等路径至少覆盖一类。

测试重点：

- 成功移动。
- avatar 不存在时拒绝。
- 非法目标时拒绝。
- 同一 tick 多个移动的处理结果稳定。

### PR 6: Event Log and Replay v0

目标：

- 建立内存事件日志和最小 replay。

完成定义：

- accepted world-changing event 被追加到 event log。
- 从初始状态 + event log 可以重建 world state。
- rejected result 不污染权威状态。

测试重点：

- replay 后 avatar 位置一致。
- 多 tick replay 顺序一致。
- rejected movement 不影响 replay state。

### PR 7: REST API v0

目标：

- 暴露最小 REST API，但 handler 不直接改权威状态。

建议 endpoint：

- `POST /api/v1/worlds`
- `POST /api/v1/intents`
- `POST /api/v1/admin/ticks`
- `GET /api/v1/observations/{agentId}`
- `GET /api/v1/events`

完成定义：

- API 只做校验、协议转换、提交 intent、查询 view。
- 状态变化仍然只发生在 world tick 中。
- 有 handler 测试覆盖成功和错误输入。

### PR 8: Store v0

目标：

- 引入持久化接口和事务边界。

完成定义：

- store interface 不污染 domain。
- 状态变更与事件写入的关系清楚。
- 可以先用内存 store 或 SQLite 证明事务语义，再决定是否接 MySQL。

测试重点：

- event 和 state 同步提交。
- 提交失败不会留下半状态。
- reload 后能恢复基本 world state。

### PR 9: Resource and Conflict v0

目标：

- 引入资源和同 tick 冲突裁决。

完成定义：

- 多个 intent 抢同一资源时排序稳定。
- 成功者产生资源事件。
- 失败者有明确拒绝原因。
- 结果可回放。

测试重点：

- 两个 avatar 抢一个资源。
- 资源不足拒绝。
- 同一输入多次运行结果一致。

### PR 10: Transaction v0

目标：

- 实现简单交易规则，验证事务一致性。

完成定义：

- 双方资源变动必须同成同败。
- 事件记录能解释交易结果。
- 失败时不会出现扣一方但未加另一方的半成功状态。

测试重点：

- 成功交易。
- 余额不足拒绝。
- 中途失败回滚。

## 4. 当前不推进的事项

暂不推进：

- agent runner。
- LLM provider adapter。
- memory / reflection / planning。
- RabbitMQ 事件总线。
- Redis 分布式锁和缓存。
- gRPC 服务拆分。
- WebSocket 实时推送。
- 多世界 / zone 分片。
- 完整 MMORPG 玩法。

这些能力不是否定项，而是必须等世界内核证明自己后再引入。

## 5. 近期判断标准

每个实现 PR 都应该优先回答：

- 这次是否让世界更像一个权威裁决系统？
- 是否保持 intent 和 event 分离？
- 是否避免绕过 `World Core` 写状态？
- 是否新增可回放的事实记录？
- 是否测试了成功路径和拒绝路径？
- 是否清楚说明哪些能力暂时不做？

如果一个 PR 只能增加接口数量，却不能增强裁决、事件、回放、事务中的至少一项，就应该推迟。

## 6. 下一步

完成本文档后，下一步应该创建 Go 工程骨架 PR。

建议 PR：

```text
backend: initialize go project skeleton
```

建议范围：

- 初始化 `go.mod`。
- 新增 `cmd/yggdrasil/main.go`。
- 新增空 package：`internal/domain`、`internal/world`、`internal/rules`、`internal/store`、`internal/api`。
- 加入最小测试，保证 `go test ./...` 可运行。

明确不做：

- 不接数据库。
- 不接 Gin。
- 不定义完整世界规则。
- 不引入 agent runner。
