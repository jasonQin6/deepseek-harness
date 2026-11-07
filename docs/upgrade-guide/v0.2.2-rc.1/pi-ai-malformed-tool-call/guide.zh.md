---
kind: upgrade-guide
description: "pi-ai 路由的格式错误工具调用改以可重试的 MALFORMED_TOOL_CALL code 失败，已存的空名调用从请求中丢弃。"
---

# 格式错误的模型工具调用使用独立错误码

[English](guide.md) | 中文

## 变更

pi-ai 路由把参数不是有效 JSON 的已完成工具调用，以及 name 为空的已完成调用，改为以新 code `MALFORMED_TOOL_CALL` 失败，而非 `MALFORMED_RESPONSE`。默认的普通重试策略会重试 `MALFORMED_TOOL_CALL`，因此重试一次该步骤通常即可清除模型采样缺陷；`MALFORMED_RESPONSE` 现在只覆盖协议违规，仍在默认可重试集合之外。两个适配器还会把已存的空名工具调用连同应答它的工具结果一起丢弃，因此本版本之前写入的 Session 会在回放时略去该调用，而其持久化内容仍保留两者。

匹配 Session 记录或 `assistant/attempt` 事件中 `failure.code` 的脚本、遥测与看板会受影响，提供方 `retryPolicy.retryableCodes` 设置同样受影响。已存的 Session 记录保留旧 code；只有新失败使用新 code。

## 迁移

1. 原先把 `MALFORMED_RESPONSE` 当作“模型返回了损坏的工具参数”的代码，改用 `MALFORMED_TOOL_CALL` 匹配。`MALFORMED_RESPONSE` 保留给格式错误的 SSE 分帧、事件类型不匹配和无法读取的回放状态，这些会确定性重复。
2. 在提供方 `retryPolicy.retryableCodes` 列表中，加入 `MALFORMED_TOOL_CALL` 以像默认策略一样重试格式错误的模型工具调用；不加入则这类尝试立即失败。
3. 验证：在会返回损坏参数的提供方上，通过 pi-ai 路由发送一个工具调用提示，确认该尝试报告 `MALFORMED_TOOL_CALL` 且该步骤被重试。然后打开升级前写入、含空名调用的 Session，确认它的下一次请求成功。
