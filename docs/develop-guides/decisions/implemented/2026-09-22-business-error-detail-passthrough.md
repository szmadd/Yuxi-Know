# 业务错误 detail 契约化与前端受控透传

状态：implemented
类型：bug-fix
Owner：web/src/apis/base.js

## 问题

后端把"业务前置校验失败"（如聊天模型不存在）与 FastAPI 请求体 schema 校验失败混用 422，detail 为纯字符串。前端 `base.js` 出于防泄密动机对所有 422 一律显示"请求参数验证失败"，并把 detail 内容替换为公共文案。用户发送消息失败时只能看到"发送消息失败: 请求参数验证失败"，无法得知真实原因。

同一掩码还产生过一个断点：`safeErrorData` 对非 422 一律返回 `{detail: 公共文案}` 字符串，使 409 dict detail 的 `code` 字段无法到达前端，`web/src/utils/toolApproval.js:87` 的 `isRunInterruptedConflict` 自 d15bedcc 引入掩码后恒为 false，`AgentChatComponent.vue:3473` 的冲突特判成为死代码。

掩码改造前的原始实现（d15bedcc^ 的 `base.js`）本就支持对象形态 detail 取 `message` 作为用户可见文案，并把完整 `errorData` 交给调用方；后端 `run_busy` 等 dict detail 先例与该消费方均早于掩码改造。本决策是恢复既有契约并收窄到安全子集，不是引入新机制。

## 决策

后端业务错误沿用既有 dict detail 先例（`run_busy`、`resume_superseded`、`queue_conflict`）并收敛为显式契约：凡 detail 为对象且含字符串 `code` 与字符串 `message` 字段时，`message` 必须是已脱敏、可直接展示给用户的文案。首个迁移点是 `agent_run_service.resolve_agent_run_model_spec` 的模型解析失败，code 为 `chat_model_not_found`，状态码保持 422，其余 raise 点不批量迁移。

前端 `base.js` 的 `extractBusinessErrorDetail` 按形态区分：detail 为数组（FastAPI schema 校验，可能回显敏感输入）继续掩码；detail 为符合契约的对象时，无论状态码，仅透传 `code` 与 `message` 两个字符串字段（其余字段丢弃），并以该 `message` 作为抛出 Error 的展示文案。字符串 detail 与其他形态维持掩码不变。这同时恢复了 `isRunInterruptedConflict` 对 409 `run_interrupted` 的识别。

## 替代方案

- 后端业务错误改用其他状态码（400/409）：语义更纯，但全仓 50+ 处 raise 422 且大量测试断言状态码，逐点迁移成本高；且 `safeErrorData` 对非 422 同样掩码，单独改状态码不能解决展示问题。
- 前端对所有字符串 detail 直接放行：改动最小，但存在 `detail=str(exc)` 透传底层异常的写法，违背 `base.js` 已声明的防泄密不变量，拒绝。
- 前端按 code 白名单放行：需要维护中央白名单清单，与"不在代码之外建立平行主张清单"的约定冲突；形态契约已由后端写代码时显式声明，白名单是重复事实源，拒绝。
- 在聊天组件内特判 `/api/agent/runs` 的 422：绕过 API 层统一安全策略，违反"API 调用统一放在 src/apis"的边界，拒绝。

## 后果

- `error.response.data.detail` 现在可能是 `{code, message}` 对象。全仓存在 `error?.response?.data?.detail || error?.message` 模式的消费方（如 `QuerySection.vue:265`、`SkillCardList.vue:856` 等约 10 处），detail 为 truthy 对象时会短路掉 `error.message` 并把对象拼进展示文案（显示 `[object Object]`）。当前不构成回归：dict detail 只出现在 agent run/queue/compression 三类端点，其消费方走 `error.message` 或 `detail.code`，不读 detail 文本。后续为这些组件对应端点新增 dict detail 时，必须先把消费方改为 `error.message` 优先（`FileUploadModal.vue:668-670` 是正确写法范例）。
- 透传不区分状态码。当前 FastAPI 默认 5xx 为纯文本非 JSON，`response.json()` 解析失败仍走掩码路径，无现实泄露面；若未来引入 5xx JSON 错误体，其 detail 不受本契约保护，需另行评估。
- `message` 是用户可见面，后端新增 dict detail 时必须保证已脱敏；该约束无法静态强制，依赖 code Review 与 `base.js` 注释。

## 验证

| 验收主张 | 失败面 | 语义 Owner | 直接证据 / 命令 | 负向案例 | 当前结果 |
|---|---|---|---|---|---|
| 模型不可用时用户看到真实原因 | detail 仍是字符串导致前端继续显示"请求参数验证失败" | `agent_run_service.resolve_agent_run_model_spec` | `docker compose exec api uv run --group test pytest test/unit/services/test_agent_run_service.py`（56 passed） | 还原字符串 detail 后 3 个 model_spec 测试按 detail 形态断言变红（实际执行：3 failed） | Passed |
| 契约形态 detail 仅透传 code/message 并作为展示文案 | 整个 detail 原样透传，或 message 未进入 Error.message | `web/src/apis/base.js` | `pnpm run test:unit`（354 passed），`api_boundary.test.js` 新增用例断言多余字段被丢弃且日志不含敏感字段 | 还原 base.js 后透传用例变红（实际执行：2 failed），数组 detail 掩码用例保持绿 | Passed |
| `isRunInterruptedConflict` 恢复识别 409 run_interrupted | 409 dict detail 仍被替换为公共文案字符串 | `web/src/apis/base.js`、`web/src/utils/toolApproval.js` | 前端 unit：409 + `{code:'run_interrupted',message}` 断言识别为 true | detail 缺 code 或 message 非字符串时返回 false（新增掩码用例覆盖） | Passed |
| 既有调用方不回归 | 掩码策略放宽影响其他错误展示 | `web/src/apis/base.js` 全部调用方 | `pnpm run lint:check`、`pnpm run test:unit`、`pnpm run build`、`python scripts/verify_engineering_contracts.py` 全部通过；后端全量 `pytest test/unit -m "not slow"` 2221 passed，1 个 sandbox provisioner 测试失败但单跑通过、与改动无 import 关系，为既有 flaky | 既有 401/423/422 数组掩码用例保持绿色 | Passed |
| 真实页面触发 422 后 toast 展示真实原因 | unit 通过但浏览器链路（如消息组件截断）失真 | `AgentChatComponent.vue` 发送失败路径 | 用户在真实页面向配置失效模型的 Agent 发送消息，toast 显示"发送消息失败: 未找到可用聊天模型: 'ark-coding:kimi-k2.6'" | — | Passed |
