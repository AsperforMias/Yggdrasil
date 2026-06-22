# Yggdrasil Agent 协作 PR 规则

本文档约束人和 coding agent 如何通过 PR 协作推进 Yggdrasil。参考 ScriptForge 的 PR 组织方式，本项目采用“小切片、强说明、可验证、文档同步”的协作模型。

## 1. 协作目标

Yggdrasil 的 PR 不追求一次性完成大系统，而是持续把“权威世界裁决服务器”拆成可验证的小块。

每个 PR 都应该让 reviewer 迅速判断：

- 这次改变了什么。
- 为什么现在要做。
- 实现边界在哪里。
- 如何验证它没有破坏世界权威路径。
- 哪些事情刻意没有做。

## 2. 分支命名

默认使用短而明确的分支名：

- `docs/landing-boundaries`
- `docs/pr-collaboration-rules`
- `backend/world-core-skeleton`
- `backend/intent-ingress`
- `backend/tick-loop-v0`
- `backend/event-log-store`
- `backend/movement-rule`
- `backend/snapshot-restore`
- `test/world-core-replay`

如果由 Codex 或其他 agent 主导实现，可以使用：

- `codex/docs/landing-boundaries`
- `codex/backend/tick-loop-v0`
- `codex/test/replay-coverage`

分支名必须表达变更范围，避免使用 `update`、`fix-stuff`、`final`、`try-2` 这类不可审查名称。

## 3. PR 标题

标题使用：

```text
<scope>: <imperative summary>
```

推荐 scope：

- `docs`
- `backend`
- `api`
- `world`
- `rules`
- `store`
- `test`
- `ops`
- `chore`

示例：

```text
docs: define landing boundaries and pr workflow
backend: add world core tick loop skeleton
world: add movement intent adjudication
store: persist event log with transaction boundary
test: add replay coverage for movement events
```

## 4. PR 正文模板

每个 PR 必须使用以下结构：

```markdown
## Summary
- ...

## Why
- ...

## Implementation
- ...

## Test
- ...

## Notes
- ...
```

这是硬约束。即使是文档 PR，也必须保留这些小节。

### 4.1 Summary

说明变更结果，不写过程流水账。

好例子：

- add the first in-memory `World Core` tick loop
- define `Intent`, `Event`, and `Observation` domain contracts
- persist accepted movement events and rejected movement results

弱例子：

- update files
- adjust code
- continue backend work

### 4.2 Why

说明为什么现在需要这个 PR，以及它和 Yggdrasil 主线的关系。

必须连接到至少一个主线目标：

- 世界状态唯一权威。
- intent 和 event 分离。
- tick 内统一裁决。
- 事件可回放。
- 事务一致性。
- observation 是局部视图。
- 当前阶段不做 agent runner。

### 4.3 Implementation

说明主要实现点和边界。需要写清楚：

- 新增了哪些模块或类型。
- 哪条写路径被改变。
- 哪些逻辑仍然是 stub 或后置项。
- 是否改变 API、schema、存储结构或测试 fixture。

如果涉及权威状态，必须说明状态如何进入 `World Core`，以及事件如何产生。

### 4.4 Test

必须列出实际运行过的验证命令或明确说明未运行原因。

可接受：

```markdown
## Test
- `go test ./...`
- `go test ./internal/world -run TestMovementIntent`
```

文档 PR 可接受：

```markdown
## Test
- documentation-only change
- manually reviewed links from `README.md`
```

不可接受：

```markdown
## Test
- should work
- not needed
```

如果测试未运行，必须写具体原因，例如：

- `not run; documentation-only change`
- `not run; Go module has not been initialized yet`
- `not run; local MySQL dependency is not available in this branch`

### 4.5 Notes

记录 reviewer 需要知道但不属于 Summary 的信息：

- 没有依赖变化。
- 没有 API 行为变化。
- 没有运行时代码变化。
- 当前限制。
- 已知后续工作。
- 对旧数据或 fixture 的影响。

## 5. PR 必填检查清单

PR 描述中必须能回答这些问题：

- 是否改变世界权威状态？
- 是否新增或改变 intent / event / snapshot / observation 的语义？
- 是否新增绕过 `World Core` 的写入路径？
- 是否涉及事务边界？
- 是否需要更新 `docs/landing-boundaries.md` 或 `docs/design/world-server-design.md`？
- 是否新增测试覆盖成功路径和拒绝路径？
- 是否有未运行测试，原因是什么？

对于纯文档或仓库维护 PR，可以明确写：

- `no runtime behavior changes`
- `no API contract changes`
- `no storage schema changes`

## 6. Agent 执行规则

Agent 接任务后应按以下顺序工作：

1. 先读 `README.md`、`docs/product-position.md`、`docs/design/world-server-design.md`、`docs/landing-boundaries.md`。
2. 判断任务属于文档、world core、rule、store、API、test、ops 中哪一类。
3. 只修改任务必要范围，避免顺手重构。
4. 修改权威状态路径前，先确认 intent、event、transaction、replay 的关系。
5. 先同步相关文档，再做行为变化；如果代码行为没有变，文档 PR 必须明确说明。
6. 运行与变更范围匹配的测试。
7. 在最终说明中列出改动文件、验证结果和未完成边界。

Agent 不应该：

- 把 Yggdrasil 改成普通 CRUD 后端。
- 在 handler / worker 里绕过世界核心直接写权威状态。
- 为了展示 agent 能力而引入 LLM runner。
- 一次 PR 同时改 API、store、规则、前端、部署，除非有明确必要。
- 用没有测试的“大重构”替代小切片推进。

## 7. 推荐 PR 类型

### 7.1 Docs Baseline PR

适合建立约束、同步方向、修正阶段目标。

要求：

- 写清为什么文档需要调整。
- 明确没有 runtime / API / schema 变化。
- 如果改变阶段目标，要同步所有相关文档入口。

### 7.2 World Core PR

适合新增 tick loop、intent queue、rule dispatch、event emission。

要求：

- 测试成功裁决和拒绝裁决。
- 明确单写路径。
- 不把外部慢调用放进裁决循环。

### 7.3 Rule PR

适合新增移动、资源、交易、任务等规则。

要求：

- 有前置条件校验。
- 有 accepted / rejected 结果。
- 有事件。
- 有边界测试，例如资源不足、目标不存在、同 tick 冲突。

### 7.4 Store PR

适合新增持久化、事务、事件日志、快照。

要求：

- 写清事务边界。
- 写清状态和事件是否同事务提交。
- 有失败路径测试或至少有明确模拟。

### 7.5 API PR

适合新增 REST endpoint。

要求：

- handler 只做认证、校验、协议转换、提交 intent。
- 不在 handler 中直接修改世界权威状态。
- 测试请求校验、成功提交、错误响应。

## 8. 示例 PR 正文

```markdown
## Summary
- add an in-memory `World Core` that accepts queued intents and processes them on manual ticks
- introduce `Intent`, `Event`, `TickResult`, and `Rule` contracts
- add movement accepted/rejected events for the first rule slice

## Why
- Yggdrasil needs a single authority path before adding storage, MQ, or agent-facing complexity
- this keeps intent separate from event and gives later PRs a stable裁决入口

## Implementation
- add `internal/world` with a tick loop and rule dispatcher
- add `internal/domain` types for avatar position, movement intent, and world events
- keep storage in memory for this PR; persistence is left for a follow-up store PR
- keep API out of scope so the world core can be tested directly first

## Test
- `go test ./internal/world ./internal/domain`

## Notes
- no API contract changes
- no database schema changes
- replay is not implemented yet; this PR only emits ordered events needed by replay
```

## 9. Merge 标准

PR 合并前至少满足：

- PR 正文完整。
- 变更范围和标题一致。
- 没有无关格式化或大面积重排。
- 文档与代码方向一致。
- 测试结果可信。
- 对未做事项有明确 Notes。

如果 PR 触碰权威状态路径，还必须满足：

- 没有绕过 `World Core` 的写入。
- intent 和 event 没有混用。
- 状态变化和事件记录关系清楚。
- 失败路径不是静默失败。

## 10. Review 关注点

Review 时优先看风险，而不是优先看风格：

- 是否破坏世界唯一权威。
- 是否让状态变化不可回放。
- 是否引入不清晰的并发写入。
- 是否把 Redis / MQ 当成最终正确性来源。
- 是否把 agent runner 混进世界内核。
- 是否缺少拒绝路径或冲突路径测试。
- 是否文档说的是 A，代码做的是 B。

风格问题只有在影响可维护性、可读性或后续边界时才阻塞。

## 11. 与落地界限的关系

`docs/landing-boundaries.md` 定义“做什么和不做什么”，本文档定义“如何通过 PR 做”。

两份文档需要一起维护：

- 边界改变时，更新落地界限。
- 协作方式改变时，更新本文档。
- PR 模板或验收口径改变时，两边都要检查是否冲突。
