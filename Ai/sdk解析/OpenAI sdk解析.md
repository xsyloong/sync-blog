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

2. SDK 定义
```ts
/**
 * ChatCompletionChunk — 流式响应的单个数据帧
 * 对应 OpenAI API object: "chat.completion.chunk"
 */
export interface ChatCompletionChunk {
  /** 
   * 本次请求的唯一标识符
   * 同一次流式请求的所有 chunk 共享同一个 id
   * 格式：chatcmpl-xxxxxxxxxxxxxxxxxxxxxxxxx
   */
  id: string;

  /**
   * chunk 中的候选回复列表
   * 通常只有一个元素（index=0），除非设置了 n > 1（多路并行生成）
   */
  choices: Array<ChatCompletionChunk.Choice>;

  /**
   * chunk 创建时的 Unix 时间戳（秒）
   */
  created: number;

  /**
   * 使用的模型名称
   * 例如：gpt-4o、gpt-4o-mini、gpt-4-turbo
   */
  model: string;

  /**
   * 固定值 "chat.completion.chunk"
   * 用于与非流式响应（"chat.completion"）区分
   */
  object: 'chat.completion.chunk';

  /**
   * 服务端系统指纹，反映服务端配置状态
   * 相同 seed + 相同 fingerprint = 可复现的输出
   * 可选字段，并非每个 chunk 都有
   */
  system_fingerprint?: string;

  /**
   * Token 用量统计
   * 注意：只有最后一个 chunk（finish_reason 不为 null 时）才携带此字段
   * 需要在请求中设置 stream_options: { include_usage: true } 才会返回
   */
  usage?: CompletionUsage;
}
```

```ts
namespace ChatCompletionChunk {
  export interface Choice {
    /**
     * 增量内容 —— 每个 chunk 携带的"新增"部分
     * 这是流式与非流式最核心的区别：
     *   非流式：message（完整内容）
     *   流式：delta（增量内容）
     * 
     * 将所有 chunk 的 delta 拼接起来 = 完整的 message
     */
    delta: ChoiceDelta;

    /**
     * 结束原因 —— 标志当前 choice 是否已结束
     * 
     * null          → 该 choice 还在生成中（中间 chunk）
     * "stop"        → 正常结束（遇到 stop token 或达到自然结束）
     * "length"      → 达到 max_tokens 限制被截断
     * "tool_calls"  → 模型决定调用工具（Function Calling）
     * "content_filter" → 内容被安全过滤器拦截
     */
    finish_reason: 'stop' | 'length' | 'tool_calls' | 'content_filter' | null;

    /**
     * 候选结果索引
     * 当 n=1 时永远是 0
     * 当 n=3 时会有 index 0、1、2 三个 choice 交替出现在不同 chunk 中
     */
    index: number;

    /**
     * 对数概率信息（需显式请求 logprobs: true）
     * 用于分析模型对每个 token 的置信度
     */
    logprobs?: ChoiceLogprobs | null;
  }
}
```

```ts
export interface ChoiceDelta {
  /**
   * 文本增量内容
   * 
   * 生命周期行为：
   * - 第一个 chunk：可能是空字符串 ""（角色声明帧）
   * - 中间 chunk：正常文本片段，如 "Hello"、" World"、"！"
   * - 最后一个 chunk：content 为 null（finish_reason 变为非 null）
   * 
   * 当模型在调用工具（tool_calls）时，content 为 null
   */
  content?: string | null;

  /**
   * 消息角色
   * 
   * 只在第一个 chunk 中出现，值为 "assistant"
   * 后续所有 chunk 此字段为 undefined（不重复传输，节省带宽）
   */
  role?: 'system' | 'user' | 'assistant' | 'tool';

  /**
   * 工具调用增量（Function Calling / Tool Use）
   * 当 finish_reason === "tool_calls" 时出现
   * 详见下方 ToolCall 结构解析
   */
  tool_calls?: Array<ChoiceDeltaToolCall>;

  /**
   * @deprecated 旧版 Function Calling（已被 tool_calls 取代）
   * 仍保留用于向后兼容
   */
  function_call?: ChoiceDeltaFunctionCall;
}
```

```ts
export interface ChoiceDeltaToolCall {
  /**
   * 工具调用在数组中的索引
   * 模型可以并行调用多个工具，通过 index 区分属于哪个工具的增量
   */
  index: number;

  /**
   * 工具调用唯一 ID
   * 只在该工具调用的第一个 chunk 中出现
   * 格式：call_xxxxxxxxxxxxxxxxxxxxxxxxx
   * 后续 chunk 此字段为 undefined
   */
  id?: string;

  /**
   * 工具类型，目前只有 "function"
   */
  type?: 'function';

  /**
   * 函数调用的增量信息
   */
  function?: {
    /**
     * 函数名称
     * 只在该工具第一个 chunk 出现，后续为 undefined
     */
    name?: string;

    /**
     * 函数参数的 JSON 字符串增量
     * 注意：这是增量！需要拼接所有 chunk 才能得到完整 JSON
     * 完整拼接后才能 JSON.parse()，中途 parse 会报错
     * 
     * 例如分三个 chunk 传输 {"city": "Beijing"}：
     * chunk1.arguments = '{"city"'
     * chunk2.arguments = ': "Bei'
     * chunk3.arguments = 'jing"}'
     */
    arguments?: string;
  };
}
```

|属性路径|类型|出现时机|说明|
|---|---|---|---|
|`id`|`string`|所有 chunk|同一请求所有 chunk 共享|
|`object`|`string`|所有 chunk|固定为 `chat.completion.chunk`|
|`model`|`string`|所有 chunk|实际使用的模型名|
|`choices[0].delta.role`|`string`|仅第一个 chunk|固定为 `assistant`|
|`choices[0].delta.content`|`string \| null`|文本生成中|文本增量，工具调用时为 `null`|
|`choices[0].delta.tool_calls`|`array`|工具调用时|工具调用增量数组|
|`choices[0].delta.tool_calls[i].index`|`number`|工具调用时|区分并行多工具|
|`choices[0].delta.tool_calls[i].id`|`string`|工具首帧|工具调用唯一 ID|
|`choices[0].delta.tool_calls[i].function.name`|`string`|工具首帧|函数名，只出现一次|
|`choices[0].delta.tool_calls[i].function.arguments`|`string`|工具调用中|参数 JSON 增量，需拼接|
|`choices[0].finish_reason`|`string \| null`|最后一帧|`null`=生成中，非null=结束|
|`usage`|`object`|最末帧|需开启 `include_usage`|
