1. Chunk定义
```text
传统请求模式（非流式）：

  Client ──────── Request ────────► Server
  Client ◄─── 等待全部生成完毕 ──── Server
  Client ◄─────── Full Response ─── Server
  
  用户体验：长时间白屏等待 ❌

流式模式（Streaming）：

  Client ──── Request ───────────────────────────────► Server
  Client ◄── chunk1 ◄── chunk2 ◄── chunk3 ◄── [DONE] ── Server
  
  用户体验：字符逐步出现，类似打字机效果 ✅
```
基于SSE协议实现，chunk其实就是sse协议的数据帧

> [!info] sse
> **SSE 协议格式**（底层传输）：
> 
>data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[...]}
>
>data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[...]}
>
>data: [DONE]
>
>每个 `data:` 行就是一个 Chunk 的原始 JSON，`[DONE]` 标志流结束。SDK 将这层解析封装掉了，开发者直接拿到反序列化后的对象。
> 