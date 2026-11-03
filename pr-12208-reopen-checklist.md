# PR #12208 重开清单 — fix(web): restore an in-flight turn after a reload

基线：commit 19b1c67（已关闭，未合并）。所有改动都在 `web/src/contexts/AgentContext.tsx`。
重开前：把每个 🔴 修复对应到 IftekharUddin 的 blocking 评论并回复/resolve。

## 🔴 Blocking（code owner 明确要求当场修）

### 1. 快照绑定 turn 身份（修 "Old output appears on new turn"）
- 快照存储键从 `sessionId` 改为 `sessionId + turn_id`（或存 `turn_started_at`）
- 恢复时校验服务器当前 turn 身份，不匹配 → 丢弃快照，走正常 hydration
- 对应讨论：discussion_r4449314276

### 2. 轮询失败保留恢复输出（修 "Network failure ends recovery"）
- `finalRows === null` 时：不清空 `setStreamingContent/Thinking`，保留快照
- 增加重试（指数退避，有上限），或至少渲染显式错误 + 手动 retry 按钮
- 对应讨论：discussion_r4449314242

### 3. Clear 取消/隔离 detached 恢复（修 "Recovery overwrites cleared chat"）
- `resetTranscriptState()` 中：置 `awaitingDetachedTurn = false` + 让 polling effect 感知 transcript 被重置（generation token）
- polling 落地写回前校验 generation，已失效则丢弃结果
- 对应讨论：discussion_r4449314252

### 4. Stop 路径绑定会话所有权（修 "Stop replaces another conversation"）
- 1451–1458 两个 await 之后各加一次 `activeSessionIdRef.current === sessionId` 校验
- 不匹配 → 不触碰视图状态
- 对应讨论：discussion_r4449314265

## 🟡 非 blocking 但建议一起修

### 5. turn 恰好在 getMessages 与 getState 之间结束（修 "Finished answer stays missing"）
- state 显示 idle 且快照存在但 transcript 未含 turn 后内容时，重新拉一次 transcript 再决定删快照
- 对应讨论：discussion_r4449314229

### 6. unmount 时保存最新快照（修 "Leaving chat loses partial output"）
- cleanup 函数里（取消计时器前）先做一次同步快照写入，不只是 `pagehide`
- 对应讨论：discussion_r4449314269

## 验证要求（对齐原 PR 模板）

- 新增/更新测试：`web/src/contexts/sessionLifecycle.test.ts`
  - Clear 与在飞轮询并发
  - Stop 延迟响应跨会话
  - 快照 turn 身份不匹配
  - transcript 读取失败后的重试
- `npx tsc -b && npm run build` 通过
- 手动场景：
  1. 长 turn（sleep 10）中途刷新 → prompt + 部分输出保留，最终答案自动落地
  2. Clear 后轮询 tick 落地 → 视图不被覆盖
  3. A 会话 Stop → 切 B → A 延迟响应不污染 B
  4. 断网 6 次请求失败后恢复网络 → 最终答案仍出现
- 重开说明里注明：修复了 Greptile 6 项 + IftekharUddin 3 项 blocking，引用对应 discussion ID

## 关联

- Blocking issue: #11517 / #7759（服务端生命周期，另案）
- 重叠工作：#12107（socket 重开场景）、草稿 #9002（服务端断连存活）
- 重开前在 PR 里说明与 #12107 的边界，避免双头重复实现
