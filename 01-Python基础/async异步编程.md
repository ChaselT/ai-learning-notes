---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# async异步编程

## 这是什么/为什么重要

LLM 应用是典型的 IO 密集场景：一次 API 调用要等几秒，流式输出要持续收 token，RAG 要并发查多个数据源。受 GIL 限制（见[[Python与Java核心差异]]），Python 的答案不是多线程而是 **asyncio**：单线程事件循环 + 协程。类比 Java：`async/await` 的书写体验像同步代码，语义接近 `CompletableFuture` 的编排，调度模型则神似虚拟线程（海量廉价任务、遇 IO 让出）。FastAPI、openai SDK、LangChain 全面提供 async API，绕不开。

## 核心内容

### 事件循环：单线程轮流跑协程

心智模型：一个线程上有一个调度器（event loop），管理成千上万个协程（coroutine）。协程遇到 `await`（IO 等待）就**主动让出**，循环切去跑别的协程——并发来自「等待时间的重叠」，不是多核并行。

- 对比 Java 线程：抢占式、由 OS 调度、栈开销大；协程：协作式、由事件循环调度、开销极小
- 最像的 Java 概念：虚拟线程（Loom）——也是海量轻量任务挂在少数载体线程上

### async / await 基础

```python
import asyncio

async def fetch_data(name: str, delay: float) -> str:
    print(f"{name} start")
    await asyncio.sleep(delay)       # 模拟 IO：让出控制权，绝不能用 time.sleep！
    print(f"{name} done")
    return f"{name}-result"

async def main() -> None:
    result = await fetch_data("a", 1.0)   # await：挂起当前协程直到完成
    print(result)

asyncio.run(main())                  # 程序入口：创建事件循环并运行
```

关键点：

- `async def` 定义协程函数，**调用它只返回协程对象，不执行**——必须 `await` 或交给事件循环
- `await` 只能出现在 `async def` 内部
- 类比：`async def` ≈ 返回 `CompletableFuture<T>` 的方法，`await` ≈ 不阻塞载体线程的 `join()`

### asyncio.gather：并发的核心

```python
import asyncio, time

async def call_llm(prompt: str) -> str:
    await asyncio.sleep(1.0)         # 假装是一次 1 秒的 LLM API 调用
    return f"answer to: {prompt}"

async def main() -> None:
    start = time.perf_counter()
    answers = await asyncio.gather(          # ≈ CompletableFuture.allOf + 收集结果
        call_llm("q1"),
        call_llm("q2"),
        call_llm("q3"),
    )
    print(answers)                            # 结果顺序与传入顺序一致
    print(f"total: {time.perf_counter() - start:.2f}s")   # ≈ 1 秒，不是 3 秒

asyncio.run(main())
```

三个 1 秒的调用并发执行，总耗时约 1 秒。逐个 `await` 则要 3 秒——**先创建所有协程再统一 gather** 是最常用的提速手段。批量 embedding、并发调多个模型对比、RAG 多路召回，全是这个模式。

### 真实场景：httpx 异步请求

```python
import asyncio
import httpx

async def fetch(client: httpx.AsyncClient, url: str) -> int:
    resp = await client.get(url)
    return resp.status_code

async def main() -> None:
    async with httpx.AsyncClient(timeout=10) as client:   # 异步上下文管理器
        codes = await asyncio.gather(
            fetch(client, "https://example.com"),
            fetch(client, "https://httpbingo.org/get"),
        )
        print(codes)

asyncio.run(main())
```

`async with` / `async for` 是[[装饰器与上下文管理器]]中 `with` 的异步版本；流式收 LLM token 用的就是 `async for chunk in stream`。

### 为什么 LLM 应用离不开 async

1. **流式输出**：token 一个个到达，`async for` 边收边推给前端，配合 SSE 实现打字机效果（见 [[流式输出与SSE]]）
2. **并发调用**：同时调多个模型/多路检索，延迟取最大值而非总和
3. **高并发服务**：FastAPI 的 async 路由让单进程同时挂起成千上万个等 LLM 响应的请求，线程模型做不到这个密度

### 最大的坑：阻塞调用卡死事件循环

事件循环是单线程的，任何**同步阻塞调用**都会冻结所有协程：

```python
async def bad() -> None:
    time.sleep(5)                # 灾难：整个事件循环卡 5 秒，所有请求全部暂停
    requests.get(url)            # 同罪：requests 是同步库

async def good() -> None:
    await asyncio.sleep(5)                          # 正确：异步睡眠
    async with httpx.AsyncClient() as c:
        await c.get(url)                            # 正确：异步 HTTP 库
    data = await asyncio.to_thread(cpu_heavy_fn)    # 同步/CPU 密集函数丢进线程池
```

规则：async 函数里出现 `time.sleep`、`requests`、同步 DB 驱动 = 事故。要么换异步库（httpx、asyncpg），要么 `asyncio.to_thread` 包一层。这是 FastAPI 服务「越跑越慢」最常见的原因。

## 动手任务

1. 写 `E:\workspace\AiStudy\phase0-python\async_basics.py`：实现 `call_llm` 模拟函数，分别用「逐个 await」和 `asyncio.gather` 跑 5 个调用，打印两种方式的总耗时对比。
2. 写 `E:\workspace\AiStudy\phase0-python\async_fetch.py`：先 `uv add httpx`，用 `httpx.AsyncClient` + `gather` 并发请求 3 个真实 URL，打印每个的状态码与耗时；再故意把其中一个换成 `requests.get`（或 `time.sleep(3)`），观察整体耗时如何劣化，并在注释里解释原因。
3. 回到[[Python与Java核心差异]]的动手任务 3，修订你当时对「为什么用 asyncio 而不是多线程」的回答。

## 相关笔记

- [[01-Python基础-MOC]]
- [[装饰器与上下文管理器]] —— 上一篇：async with 的来源
- [[常用标准库与生态]] —— 下一篇：httpx 详细用法
- [[Python与Java核心差异]] —— GIL：为什么走到 async 这条路
- [[流式输出与SSE]] —— 阶段 1：async 的主战场
