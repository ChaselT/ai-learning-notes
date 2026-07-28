---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# 流式输出与SSE

## 这是什么/为什么重要

LLM 生成一段几百 token 的回答要好几秒甚至几十秒，如果等全部生成完再返回，用户面对的是漫长白屏。流式输出让 token 生成一个就推一个，**首字延迟（TTFT）从几秒降到几百毫秒**，这是 ChatGPT 体验流畅的关键。做任何面向用户的 LLM 产品，流式几乎是必选项。

## 核心内容

### 1. 为什么天然适合流式

LLM 是逐 token 生成的（见 [[大语言模型工作原理速览]]），服务端本来就是一个 token 一个 token 产出的——攒齐再发反而是人为增加延迟。流式只是把生成过程"直播"出来。

### 2. SSE 协议（Server-Sent Events）

Chat API 的流式传输用的是 SSE：基于普通 HTTP 的**服务端单向推送**，响应头 `Content-Type: text/event-stream`，连接保持打开，服务端不断写入形如 `data: {...}\n\n` 的文本块，最后以 `data: [DONE]` 结束。

与你熟悉的方案对比（Java 工程师视角）：

| 方案 | 方向 | 连接 | 典型场景 |
|---|---|---|---|
| 长轮询 | 客户端反复问 | 每次新请求 | 老式消息通知，开销大 |
| WebSocket | 双向 | 独立协议（ws://），需升级握手 | 聊天室、协同编辑 |
| SSE | 服务端→客户端单向 | 就是 HTTP，穿透代理友好 | LLM 流式输出、进度推送 |

LLM 场景只需要"服务端往下推"，所以 SSE 比 WebSocket 更简单合适。Spring 里对应 `SseEmitter` / WebFlux 的 `Flux<ServerSentEvent>`。

### 3. openai SDK：stream=True 逐 chunk 处理

SDK 把 SSE 解析细节封装好了，你拿到的是一个可迭代的 chunk 流：

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                base_url="https://api.deepseek.com/v1")

stream = client.chat.completions.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "写一首关于程序员的五言绝句"}],
    stream=True,
)

full_text = []
for chunk in stream:
    delta = chunk.choices[0].delta.content  # 本次增量，可能为 None（如首个角色帧/结束帧）
    if delta:
        print(delta, end="", flush=True)    # flush 立刻上屏，终端才有打字机效果
        full_text.append(delta)
print()
answer = "".join(full_text)  # 多轮对话记得把完整回复回填 history
```

要点：

- 每个 chunk 只含**增量**（`delta.content`），不是全量；需要自己拼接
- `delta.content` 可能为 `None`，必须判空
- 流式模式下 `usage` 默认拿不到，可加 `stream_options={"include_usage": True}`（最后一个 chunk 携带）
- 多轮对话中，拼接出的完整文本仍要回填进 `messages`（见 [[Chat Completions API]]）

### 4. async 流式

Web 服务里一个请求挂着几十秒的流，同步写法会占死线程，要用异步（前置知识：[[async异步编程]]）：

```python
import asyncio, os
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                     base_url="https://api.deepseek.com/v1")

async def main() -> None:
    stream = await client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": "一句话介绍SSE"}],
        stream=True,
    )
    async for chunk in stream:           # 注意是 async for
        delta = chunk.choices[0].delta.content
        if delta:
            print(delta, end="", flush=True)
    print()

asyncio.run(main())
```

FastAPI 中把这个 async generator 接到 `StreamingResponse` 上，就是一个流式聊天接口（见 [[FastAPI入门]]）。

### 5. 流式的工程代价

- 错误处理更麻烦：非流式失败就是一个异常，流式可能"输出一半断了"，要决定重试策略（整体重来？把已收内容拼进 prompt 续写？）
- 结构化输出与流式天然冲突：JSON 没收完就是不合法的（见 [[结构化输出]]）
- 中间层（Nginx 等）需关闭响应缓冲（`X-Accel-Buffering: no`），否则流被攒成一坨一次性吐出，流式白做
- 内容审核变难：不良内容可能已经流给用户才被识别，严格场景需逐段审核或放弃流式

两个衡量指标记住名字，做性能优化时会反复用到：**TTFT**（Time To First Token，首字延迟，流式优化的就是它）和 **TPS**（tokens per second，生成吞吐，决定"打字"速度）。

选型结论：面向人的对话界面用流式；面向程序的结构化调用（抽取、分类、工具决策）用非流式，简单可靠。

## 动手任务

1. 写 `E:\workspace\AiStudy\phase1-llm-api\ex05_stream_basic.py`：流式打印回答，并统计首 chunk 延迟和总耗时（`time.perf_counter()`），与非流式对比 TTFT
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex05_stream_chat.py`：把 ex03 的多轮对话改造成流式版，确认回填历史后记忆正常
3. 写 `E:\workspace\AiStudy\phase1-llm-api\ex05_stream_async.py`：用 AsyncOpenAI + `asyncio.gather` 并发向本地 Ollama 发 3 个流式请求，体会异步并发

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[大语言模型工作原理速览]]
- [[Chat Completions API]]
- [[async异步编程]]
- [[FastAPI入门]]
- [[结构化输出]]
- [[AI学习地图]]
