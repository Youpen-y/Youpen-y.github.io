---
title: "从 Prompt 到 Agent：OpenAI API 演进路线"
date: 2026-05-30T21:39:50+08:00
draft: false
tags: ["ai", "api"]
comments: true
---
> **API 设计始终以模型自身的工作原理为指导！**\
> ——[Why we built the Responses API](https://developers.openai.com/blog/responses-api) By **Steve Coffey, Prashant Mital**

2020年至2025年，OpenAI 的 API 经历了三次重大演进：从文本补全到多轮对话，再到智能体推理与行动。每一次迭代都根植于模型能力的跃升，反过来又塑造了开发者的使用方式。本文将沿着这条线索，逐一梳理三代 API 的设计理念与核心参数。

### Completions API
2020年6月11日，OpenAI 宣布推出用于访问 GPT-3 系列模型的 Completions API（`/v1/completions`）。用户只需提供提示词与参数，模型便会返回相应的文本补全（completion）。两年后的7月份，该 API 在迎来最后一次更新后，便逐步退出历史舞台。回顾这一 API，有助于我们认识当时的设计考量。

下面是API的Body Parameters：
{{% table cols="1.9:2:0.5:1.5:3:2.5" %}}
| 参数名 | 类型 | 是否必选 | 候选项 | 含义 | 注意点 |
|---|---|---|---|---|---|
| `model` | string | 是 | "gpt-3.5-turbo-instruct" / "davinci-002" / "babbage-002" | 指定使用的模型 | - |
| `prompt` | string/array of string/array of number/array of array of number | 是 | 自定义 | 用户定义的用于生成完整内容的提示词 | - |
| `best_of` | number | 否 | [0, 20] | 指定在服务器端生成补全的数目，从中返回“最优”（每 token 对数概率最高）的结果。| 结果不会流式输出；大值很费 token |
| `echo` | boolean | 否 | true/false | 除了补全，同时返回提示词 | - |
| `frequency_penalty` | number | 否 | [-2.0, 2.0] | 正数值会根据新生成的 token 在文本中出现的频率来对其施加惩罚，从而降低模型逐字重复相同内容的可能性。 | - |
| `logit_bias` | map[number] | 否 | JSON object {"token_id": value} | 调整特定符号在补全结果中出现的概率 | value 的范围: [-100, 100]，设置为 -100 则阻止生成 |
| `logprobs` | number | 否 | [0, 5] | 同时列出最有可能出现的 logprobs 个输出 token 的日志概率，以及被选中的 token | 如果 logprobs 的值为 5，那么 API 将返回 5 个最有可能出现的 token。API 总会返回 logprob 个被抽中的 token，因此响应结果中最多可能包含 logprobs+1 个元素。|
| `max_tokens` | number | 否 | [0, 模型上限] | 补全中生成的 token 上限 |（提示词 token 数 + max_tokens）<= 模型处理上限 |
| `n` | number | 否 | [1, 128] | 每个提示词需要生成的补全数量 | 大 n 费 token |
| `presence_penalty` | number | 否 | [-2.0, 2.0] | 设为正数值会根据新生成的 token 在文本中出现的频率来对其施加“惩罚”，从而增加模型讨论新主题的可能性。| - |
| `seed` | number | 否 | int64 | 指定后，系统会尽力以确定性方式来处理请求。 | 确定性并不具有绝对的保障，通过 `system_fingerprint` 响应参数来了解后端发生的各种变化。|
| `stop` | string/array of string | 否 | - | 遇到任一停止序列后，停止生成更多 token | 返回文本中不包含这些停止序列 |
| `stream` | boolean | 否 | true/false | 是否流式传输响应 | - |
| `stream_options` | object | 否 | - | 流式响应的选项 | 仅在 stream: true 时有效，设置 `{"include_usage": true}` 获取使用统计 |
| `suffix` | string | 否 | - | 在插入文本后添加的后缀 | 参数仅适用于 `gpt-3.5-turbo-instruct` 模型 | 
| `temperature` | number | 否 | [0, 2] | 采样温度 | 较高的数值，比如 0.8，会使输出结果更加随机；而较低的数值，比如 0.2，则会使输出结果更加有规律、更易于预测。 |
| `top_p` | number | 否 | [0, 1] | 核心采样：只考虑高概率值的 tokens | top_p: 0.1 意味着只考虑概率值前 10% 的 tokens |
| `user` | string | 否 | id | 标识终端用户的唯一标识符 | 帮助 OpenAI 监控和识别各种滥用行为 |
{{% /table %}}

端点响应如下（无论是流式还是非流式，结构相同）：
```js
{
    "id": "cmpl-uqkvlQyYK7bGYrRHQ0eXlWi7", // 该补全的唯一标识符
    "object": "text_completion", // 始终是 "text_completion"
    "created": 1780188213, // 补全创建时的 Unix 时间戳（以秒为单位）
    "model": "model_id", // 补全内容的模型
    "system_fingerprint": "fp_44709d6fcb", // 模型运行时所依赖的后端配置信息，可与 seed 请求参数一起使用，以便了解后端发生了哪些可能影响系统确定性的更改
    "choices": [{
        "text": "\n\nThis is indeed a test", // 补全响应的文本内容，可通过 response.choices[0].text 访问
        "index": 0, // 响应索引
        "logprobs": null, 
        /*
        "logprobs": {
            "text_offset": [0,..], // 文本偏移量
            "token_logprobs": [0,..], // 词元概率
            "tokens": ["test",..], // 各个词元
            "top_logprobs": [] // 概率最高的词元
        }
        */
        "finish_reason": "stop" // 结束原因："stop" 自然停止、"length" 达到最大 token 数、"content_filter" 内容过滤
    }],
    "usage": { // 完成请求的使用统计信息
        "prompt_tokens": 5, // 提示词中的 token 数量
        "completion_tokens": 7, // 生成的补全内容中包含的 token 数量
        "total_tokens": 12, // 该请求中使用的 tokens 总数（prompt+completion）
        "completion_tokens_details": {
            "accepted_prediction_tokens": 0, // 在使用“预测输出”功能时，出现在补全中的预测 tokens 数目
            "audio_tokens": 0, // 模型生成的音频 tokens
            "reasoning_tokens": 0, // 模型为进行推理而生成的 tokens
            "rejected_prediction_tokens": 0 // 在使用“预测输出”功能时，未出现在补全中的预测 tokens 数目
        },
        "prompt_tokens_details": { // 提示词中使用的各类 token 的详细分类
            "audio_tokens": 0, // 提示词中包含的音频输入 tokens
            "cached_tokens": 0, // 提示词中包含的缓存 tokens
        }
    }

}
```

### Chat Completions API
2023年3月，ChatGPT 席卷全球不久，OpenAI 发布了 [GPT-3.5 Turbo](https://openai.com/index/introducing-chatgpt-and-whisper-apis/) 与 Chat Completions API（`/v1/chat/completions`），随后又在6月通过 [GPT-4 API](https://openai.com/index/gpt-4-api-general-availability/?utm_source=chatgpt.com) 对其进行了大幅更新。模型的能力增强促使交互方式从单纯的文本补全演进为多轮对话，API 的设计也随之革新。

Chat Completions API 与 Completions API 最大的区别在于，使用结构化的 **messages** 替代了自由格式的 **prompt**。每条消息引入角色标识（`system` / `user` / `assistant` / `tool` / `function`），让多轮对话和指令遵循更自然可靠。由于模型推理能力增强，Chat Completions API 引入了推理强度（`reasoning_effort`）参数。同时，它还提供了工具调用能力，使模型能够与外部工具和系统进行交互，进一步扩展了大模型的应用边界，推动模型从“对话生成”向“智能体执行”演进。

下面是 API 的核心 Body Parameters（参数支持因模型而异）：

{{% table cols="1.9:2:0.5:1.5:3:2.5" %}}
| 参数名 | 类型 | 是否必选 | 候选项 | 含义 | 注意点 |
|---|---|---|---|---|---|
| `model` | string | 是 | "gpt-4o" / "gpt-4.1" / "o3" 等 | 指定使用的模型 | 文本生成、视觉、语音 |
| `messages` | array of object | 是 | 自定义 | 对话消息列表，每条包含 `role` 和 `content` | 支持多模态内容（文本 + 图片） |
| `max_tokens` | number | 否 | [1, 模型上限] | 生成的 token 上限 | 弃用状态，部分新模型已改用 `max_completion_tokens` |
| `max_completion_tokens` | number | 否 | [1, 模型上限] | 生成的 token 上限（含推理 token） | 包括可见的输出 tokens 和推理 tokens |
| `modalities` | array | 否 | array of "text"/"audio" | 希望模型输出的类型 | 默认为 `["text"]` |
| `temperature` | number | 否 | [0, 2] | 采样温度 | 较高值使输出更随机，较低值更确定 |
| `top_p` | number | 否 | [0, 1] | 核心采样，仅考虑概率排名前 p 的 token | 与 `temperature` 二选一调整即可 |
| `n` | number | 否 | [1, 128] | 为每条消息生成的回复数量 | 大 n 费 token |
| `stream` | boolean | 否 | true/false | 是否流式传输响应 | - |
| `stop` | string / array of string | 否 | 最多设置 4 个序列 | 输出遇到停止序列后停止生成 | 返回文本中不包含停止序列 |
| `store` | boolean | 否 | true/false | 是否存储此请求的输出 | 用于提供模型蒸馏服务 |
| `prediction` | object | 否 | - | 静态预测输出内容 | |
| `presence_penalty` | number | 否 | [-2.0, 2.0] | 正值惩罚已出现的 token，增加讨论新主题的可能 | - |
| `prompt_cache_key` | string | 否 | - | 缓存类似请求的响应，以优化缓存命中率 | 取代 `user` 字段 |
| `prompt_cache_retention` | string | 否 | "in_memory" 或 "24h" | 提示缓存的保存策略 | - |
| `reasoning_effort` | string | 否 | "none"/"minimal"/"low"/"medium"/"high"/"xhigh" | 限制推理模型的推理强度 | 减少推理强度可加快响应速度，减少响应中用于推理的 tokens |
| `response_format` | object | 否 | JSON Schema | 指定输出格式（如 JSON 模式） | 可配合 Structured Outputs 使用 |
| `safety_identifier` | string | 否 | 最长64个字符 | 唯一标识每个用户的字符串 | 帮助检测可能违反 OpenAI 使用政策的应用用户 |
| `frequency_penalty` | number | 否 | [-2.0, 2.0] | 正值根据频率惩罚 token，降低逐字重复的可能 | - |
| `logit_bias` | map[number] | 否 | {"token_id": value} | 调整特定 token 出现的概率 | value 范围 [-100, 100]；-100 阻止生成 |
| `logprobs` | boolean | 否 | true/false | 是否返回输出 token 的日志概率 | - |
| `top_logprobs` | number | 否 | [0, 20] | 每个位置返回概率最高的 N 个 token | 需 `logprobs: true` |
| `tools` | array of object | 否 | - | 模型可调用的工具列表 | 函数调用（Function Calling）核心参数 |
| `tool_choice` | string / object | 否 | "auto" / "required" / "none" | 控制模型如何选择工具调用 | 可指定具体函数名 |
| `parallel_tool_calls` | boolean | 否 | true/false | 是否允许并行调用多个工具 | - |
| `stream_options` | object | 否 | - | 流式响应选项 | 设置 `{"include_usage": true}` 获取使用统计 |
| `verbosity` | string | 否 | "low"/"medium"/"high" | 限制模型响应的详细程度 | 值越低，响应越简洁；值越高，响应越详细 |
| `web_search_options` | object | 否 | - | 网络搜索选项 | 搜索上下文大小与用户位置 |
{{% /table %}}

**messages** 中每条消息的核心字段：

{{% table cols="2:3:6" %}}
| 字段 | 类型 | 说明 |
|---|---|---|
| `role` | string | 消息角色：`system`（系统指令）、`user`（用户输入）、`assistant`（模型回复）、`tool`（工具返回结果） |
| `content` | string / array | 消息内容；array 形式支持混合文本和图片（多模态） |
| `name` | string | 参与者的可读名称，用于区分不同的 `user` 或 `tool` |
| `tool_calls` | array of object | 模型生成的工具调用请求（仅 `assistant` 消息），模型创建的函数工具或自定义工具 |
| `tool_call_id` | string | 对应的工具调用 ID（仅 `tool` 消息） |
| `refusal` | string / null | 模型拒绝回答时的拒绝信息 |
{{% /table %}}

端点响应如下（非流式）：
```js
{
    "id": "chatcmpl-abc123",                   // 唯一标识符
    "object": "chat.completion",               // 对象类型
    "created": 1780289625,                     // 创建时间（Unix 时间戳）
    "model": "gpt-5.4",                        // 实际使用的模型
    "choices": [    // chat completion 选项列表，如果请求参数中 n > 1，则将有多个选项
        {
            "index": 0,                        // 选项索引
            "message": {
                "role": "assistant",                // 消息角色
                "content": "Hello! How can I ...",  // 消息内容
                "refusal": null,                    // 模型生成的拒绝信息（如触发安全策略）
                "tool_calls": [],                   // 模型生成的工具调用请求（如有）
                /*
                "audio": {  // 如果请求了音频输出模态，audio 包含音频响应数据
                    "id": "string",             // 音频响应唯一标识符
                    "data": "string",           // 模型生成的 Base64 编码音频字节，格式为请求中指定格式
                    "expires_at": 1780290328,   // 服务器上过期时间
                    "transcript": "string",     // 模型生成的音频的文字稿
                }
                */
            },
            "logprobs": null,                    // 日志概率（按请求返回）
            "finish_reason": "stop"              // 结束原因：stop / length / tool_calls / content_filter
        }
    ],
    "usage": {  // 完成请求的使用统计信息
        "prompt_tokens": 9,                     // 提示词 token 数
        "completion_tokens": 12,                // 补全 token 数
        "total_tokens": 21,                     // 总 token 数
        "completion_tokens_details": {
            "accepted_prediction_tokens": 0,    // 命中的预测 token
            "rejected_prediction_tokens": 0,    // 未命中的预测 token
            "audio_tokens": 0,                   // 音频 token
            "reasoning_tokens": 0                // 推理 token（o 系列模型）
        },
        "prompt_tokens_details": {
            "audio_tokens": 0,                   // 输入音频 token
            "cached_tokens": 0                   // 缓存命中 token
        }
    }
}
```



### Responses API
Chat Completions API 取得了巨大的成功，成为事实上的行业标准。然而，随着模型在多模态理解与生成能力上的持续进步、主动调用工具的需求日益增长，以及在回复前进行深度思考等能力的涌现，其最初面向文本对话场景设计的架构开始显露出局限性。

首先，Chat Completions API 本质上是无状态的。每次请求都需要开发者自行维护完整的对话历史，并将其发送给模型。随着会话轮数增加，不仅会带来更高的 token 消耗，也增加了开发者上下文管理的复杂度。其次，虽然 Chat Completions API 已支持了函数调用和部分工具调用，但工具执行过程并非由接口原生管理。开发者需要自行解析模型返回的工具调用请求、执行工具、处理错误、并将执行结果重新封装为消息提交给模型。对于涉及多个工具、多轮推理或复杂工作流的场景，这种模式往往需要编写大量编排代码和状态管理逻辑。此外，随着推理模型的流行，模型在生成最终答案之前可能会经过复杂的内部推理，但 Chat Completions API 主要围绕 messages 进行交互，开发者无法直接管理或复用模型的推理。与此同时，模型多模态能力的发展进一步放大这些问题，文本、图片、音频、工具调用结果以及模型输出都需要映射到统一的 message 结构中，使得接口语义不断扩展，增加了应用开发的复杂性。

为此，2025年3月，OpenAI 进一步推出了 Responses API（`/v1/responses`），二者的核心差异如下：

{{% table cols="1.7:5:5" %}}

|       区别         | **Chat Completions API**              | **Responses API**                                                                  |
| -------------- | ------------------------------------- | ---------------------------------------------------------------------------------- |
| 交互模型           | 基于消息（Messages）的回合制聊天模型                | 基于响应（Response）的执行模型，支持推理、工具调用和结果生成的统一编排                                            |
| 核心抽象           | Message（消息）                           | Response（响应）与 Output Item（执行步骤）                                                    |
| 输出结构           | 返回单条 Assistant Message，工具调用作为消息的一部分返回 | 返回结构化 Output Items，可包含推理、工具调用、工具结果和最终回复等多个阶段                                       |
| 状态管理           | 无状态，每次请求需传递完整消息历史                     | `store: true` 保持状态，支持通过 `previous_response_id` 延续执行过程，自动保留上下文与推理状态                                    |
| 推理模型           | 仅返回最终结果，推理过程不可管理或复用                   | 推理过程成为一等公民，可在执行链路中持续保留和利用                                                          |
| 工具调用           | 需自行解析工具调用、执行工具并回传结果                   | 原生支持工具调用流程，并集成 Web Search、File Search、Code Interpreter、Computer Use、 Image Generation、Remote MCP 等工具 |
| 多模态支持          | 基于 Message 扩展文本、图片、音频等内容类型            | 统一处理文本、图片、音频、工具结果等多种输入输出形式                                                         |
| Agent 工作流      | 开发者负责状态维护、工具编排和执行控制                   | 面向 Agent 设计，将推理、工具调用与结果生成纳入统一执行框架                                                  |
| 开发复杂度          | 需要维护消息历史、工具调用链和上下文管理逻辑                | 减少状态管理与工具编排代码，降低 Agent 开发复杂度                                                       |
{{% /table %}}

下面是 API 的核心 Body Parameters（参数支持因模型而异）：

{{% table cols="1.9:2:0.5:1.5:3:2.5" %}}
| 参数名 | 类型 | 是否必选 | 候选项 | 含义 | 注意点 |
|---|---|---|---|---|---|
| `model` | string | 是 | "gpt-5.5" / "gpt-4o" / "o3" 等 | 指定使用的模型 | - |
| `input` | string / array of object | 是 | 自定义 | 输入内容，文本字符串或结构化输入项列表 | 支持多模态（文本 + 图片 + 文件） |
| `instructions` | string | 否 | 自定义 | 系统级指令，定义模型的行为方式 | 替代 Chat Completions 中的 system 消息 |
| `tools` | array of object | 否 | - | 模型可使用的工具列表 | 内置：web_search、file_search、code_interpreter、mcp；自定义：function |
| `tool_choice` | string / object | 否 | "auto" / "required" / "none" | 控制模型如何选择工具 | 可指定具体工具名称 |
| `max_output_tokens` | number | 否 | [16, 模型上限] | 响应中可生成的 token 上限（包括可见输出 token 和推理 token） | - |
| `max_tool_calls` | number | 否 | - | 限制模型在一次响应中可触发内置工具调用数目的最大限制 | 超过限制的工具调用请求将被忽略 |
| `parallel_tool_calls` | boolean | 否 | - | 是否允许模型并行运行工具调用 | - |
| `temperature` | number | 否 | [0, 2] | 采样温度 | 较高值使输出更随机，较低值更确定 |
| `top_p` | number | 否 | [0, 1] | 核心采样，仅考虑概率排名前 p 的 token | 与 `temperature` 二选一调整即可 |
| `previous_response_id` | string | 否 | - | 前一轮响应的 ID | 实现有状态多轮对话，无需重新发送完整历史 |
| `reasoning` | object | 否 | - | 推理模型的配置 | 可设置 `effort` 和 `summary` |
| `response_format` | object | 否 | JSON Schema | 指定输出格式 | 可配合 Structured Outputs 使用 |
| `truncation` | string | 否 | "auto" / "disabled"（默认） | 响应的截断策略 | 响应输入超过模型上下文，"auto" 自动删除对话开头的项目，"disabled" 将请求失败 |
| `store` | boolean | 否 | true/false | 是否存储响应结果 | 后续可通过 API 检索 |
| `stream` | boolean | 否 | true/false | 是否流式传输响应 | - |
| `metadata` | object | 否 | 键值对 | 附加到响应的元数据 | 最多 16 个键值对 |
| `verbosity` | string | 否 | "low" / "medium" / "high" | 限制模型响应的详细程度 | - |
{{% /table %}}



端点响应如下：
```js
{
    "id": "resp_abc123",                    // 响应唯一标识符
    "object": "response",                   // 对象类型（始终为 response）
    "created_at": 1741711372,               // 创建时间（Unix 时间戳）
    "model": "gpt-4o",                      // 使用的模型
    "status": "completed",                  // 响应状态：completed / failed / incomplete
    "completed_at": 1741476543,             // 完成时间
    "error": null,                          // 模型无法生成响应时返回的错误对象
    "incomplete_details": null,             // 说明回复不完整的原因： max_output_tokens / content_filter
    "instructions": null,                   // 插入到模型上下文中的系统消息，与 previous_response_id 一同使用时，前一响应的指令不会传递到下一个响应中
    "output": [                             // 模型生成的项目列表，可包含多个不同类型的项
        {
            "type": "message",              // 输出类型：消息（始终为 message）
            "id": "msg_abc",                // 消息唯一标识符
            "status": "completed",          // 输出消息状态：in_progress / completed / incomplete
            "role": "assistant",            // 输出消息的角色（始终为 assistant）
            "phase": "commentary",          // 标识 Assistant Output Item 当前处于执行流程的哪个阶段：commentary / final_answer
            "content": [                    // 输出消息的内容
                {
                    "type": "output_text",  // 输出内容类型（始终为 output_text）
                    "text": "Hello! How can I help?",   // 模型的输出文本
                    "annotations": []       // 标注信息（如引用来源）
                } 
                /*
                {
                    "type": "refusal",      // 拒绝的类型（始终为 refusal）
                    "refusal": "string"     // 模型拒绝的解释
                }
                */
            ]
        },
        {
            "type": "file_search_call",     // 输出类型：文件搜索工具调用
            "id": "tool_id",                // 文件搜索工具调用的唯一 ID
            "status": "searching",          // 文件搜索工具调用的状态：in_progress / searching / completed
            "queries": "search query",      // 用于搜索文件的查询语句
            "results": [
                {
                    "file_id": "abc",       // 文件唯一标识符
                    "filename": "temp.txt", // 文件名
                    "score": 0.5,           // 文件相关性得分：[0, 1]
                    "text": "some text"     // 文件中检索的文本
                }
            ]
        },
        {
            "type": "function_call",        // 输出类型：函数调用
            "id": "fc_xyz",                 // 函数调用唯一标识符
            "call_id": "call_abc",          // 模型生成的调用 ID，用于匹配工具返回结果
            "name": "get_weather",          // 函数名称
            "namespace": "somens",          // 要运行函数的命名空间
            "status": "in_progress",        // Item 状态：in_progress / completed / incomplete
            "arguments": "{\"location\":\"Tokyo\"}"  // 传递给函数的参数（JSON 字符串）
        }
    ],
    "parallel_tool_calls": true,            // 并行工具调用
    "previous_response_id": null,           // 模型先前响应的唯一 ID，使用此 ID 来创建多轮对话。
    "reasoning": {
        "effort": null,
        "summary": null
    },
    "store": true,
    "temperature": 1.0,                     // 采样温度取值范围
    "text": {
        "format": {
            "type": "text"
        }
    },
    "tool_choice": "auto",                  // 模型在生成响应时应如何选择要使用的工具：none / auto / required
    "tools": [],
    "top_p": 1.0,
    "truncation": "disabled",
    "usage": {                              // 使用统计
        "input_tokens": 50,                 // 输入 token 数
        "output_tokens": 100,               // 输出 token 数
        "total_tokens": 150,                // 总 token 数
        "input_tokens_details": {
            "cached_tokens": 0              // 缓存命中 token
        },
        "output_tokens_details": {
            "reasoning_tokens": 0           // 推理 token
        }
    }
}
```

### 小结

从 Completions API 的文本补全，到 Chat Completions API 的结构化对话，再到 Responses API 的推理与行动循环的三代 API 的演进可以看出，大语言模型正从"工具"走向"智能体"。每一次 API 的重新设计，都紧跟模型能力的跃迁，将新的可能性以最自然的方式暴露给开发者。未来 API 是否会迎来进一步改变呢？还是开篇所引用的那句话：**API 设计始终以模型自身的工作原理为指导。**

### 参考文章
{{< embed url="https://x.com/athyuttamre/status/1899541474297180664" title="Introducing the Responses API: the new primitive of the OpenAI API." card="true">}}

{{< embed url="https://developers.openai.com/blog/responses-api" title="Why we built the Responses API" card="true">}}

{{< embed url="https://simonwillison.net/2025/Mar/11/responses-vs-chat-completions/" title="OpenAI API: Responses vs. Chat Completions" card="true">}}

{{< embed url="https://developers.openai.com/api/reference/resources/completions/methods/create" title="Create completion" card="true">}}

{{< embed url="https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create" title="Create chat completion" card="true">}}

{{< embed url="https://developers.openai.com/api/reference/resources/responses/methods/create" title="Create a model response" card="true">}}