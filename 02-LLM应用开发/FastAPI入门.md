---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# FastAPI入门

## 这是什么/为什么重要

FastAPI 是 Python 生态里最主流的 Web 框架，AI 服务的事实标准——几乎所有 LLM 应用的 HTTP 层都用它。对你这个 Spring Boot 老手来说，概念一一对应，学习成本极低：装饰器路由、pydantic 校验、依赖注入、自动 API 文档，全都有 Spring 的影子。学完它，你就能把前面所有 LLM 能力包装成服务对外提供。

## 核心内容

### 1. 概念对照表（Spring Boot → FastAPI）

| Spring Boot | FastAPI | 说明 |
|---|---|---|
| `@RestController` + `@GetMapping` | `@app.get("/path")` | 路由装饰器 |
| `@RequestBody` + `@Valid` DTO | pydantic `BaseModel` 入参 | 自动解析+校验+400 |
| `@Autowired` / 构造器注入 | `Depends()` | 依赖注入 |
| springdoc / Swagger | 内置 `/docs` | 零配置自动生成 |
| 内嵌 Tomcat | uvicorn (ASGI 服务器) | 启动方式 |
| Filter / Interceptor | middleware / Depends | 横切逻辑 |

### 2. 最小应用 + pydantic 入参校验

当前版本基线（2026-07）：FastAPI 已经完全切到 **pydantic v2**（要求 `pydantic >= 2.7`，内部不再有 `pydantic.v1` 兼容导入），底层 Starlette 到了 1.0。你在阶段 0 学的 pydantic v2 写法（`model_validate` / `model_dump` / `Field`）在这里直接可用；网上搜到 `.dict()`、`.parse_obj()`、`@validator` 的老教程是 v1 语法，别抄。

```python
# uv add "fastapi[standard]" openai "httpx[socks]"
# main.py
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="LLM Chat Service")

class ChatRequest(BaseModel):            # 相当于 @Valid 的 DTO
    message: str = Field(min_length=1, max_length=4000)
    temperature: float = Field(default=0.7, ge=0, le=2)

class ChatResponse(BaseModel):
    reply: str
    model: str

@app.get("/health")                      # GET 路由
def health() -> dict:
    return {"status": "ok"}

@app.post("/chat", response_model=ChatResponse)   # POST + 出参模型
def chat(req: ChatRequest) -> ChatResponse:
    # 校验不通过时 FastAPI 自动返回 422 + 错误详情，无需手写判断
    return ChatResponse(reply=f"echo: {req.message}", model="stub")
```

启动与文档：

```powershell
uvicorn main:app --reload --port 8000   # --reload 类似 spring-boot-devtools 热重载
# 浏览器打开 http://localhost:8000/docs  -> 自动生成的 Swagger UI，可直接试调
```

### 3. 依赖注入 Depends

```python
from functools import lru_cache
from fastapi import Depends
from openai import AsyncOpenAI
import os

@lru_cache                                # 单例，类比 Spring 默认 singleton scope
def get_llm_client() -> AsyncOpenAI:
    return AsyncOpenAI(
        base_url=os.environ.get("LLM_BASE_URL", "http://localhost:11434/v1"),
        api_key=os.environ.get("LLM_API_KEY", "ollama"),
    )

@app.post("/chat2")
async def chat2(req: ChatRequest,
                client: AsyncOpenAI = Depends(get_llm_client)) -> dict:
    resp = await client.chat.completions.create(
        model=os.environ.get("LLM_MODEL", "qwen3.5:27b"),
        messages=[{"role": "user", "content": req.message}],
        temperature=req.temperature,
    )
    return {"reply": resp.choices[0].message.content}
```

`Depends` 比 `@Autowired` 更轻：不需要容器扫描，就是个可组合的函数调用，还能用于鉴权、获取当前用户、数据库会话等（类比 `HandlerMethodArgumentResolver`）。

### 4. 流式聊天接口（SSE）

把 [[流式输出与SSE]] 的 async 流式接到 `StreamingResponse` 上，前端（或 curl）即可逐字接收：

```python
from fastapi.responses import StreamingResponse

@app.post("/chat/stream")
async def chat_stream(req: ChatRequest,
                      client: AsyncOpenAI = Depends(get_llm_client)):
    async def event_gen():
        stream = await client.chat.completions.create(
            model=os.environ.get("LLM_MODEL", "qwen3.5:27b"),
            messages=[{"role": "user", "content": req.message}],
            stream=True,
        )
        async for chunk in stream:
            delta = chunk.choices[0].delta.content
            if delta:
                yield f"data: {delta}\n\n"     # SSE 帧格式
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_gen(), media_type="text/event-stream")
```

用 curl 验证（`-N` 关闭缓冲才能看到流式效果）：

```powershell
curl -N -X POST http://localhost:8000/chat/stream -H "Content-Type: application/json" -d "{\"message\": \"写一首七言绝句\"}"
```

注意：路由函数用 `async def` + `AsyncOpenAI`，一个 worker 就能扛住大量并发挂起的流（前置知识：[[async异步编程]]）；同步写法会阻塞事件循环，是 FastAPI 最经典的性能事故。

### 5. 错误处理与并发调用

**把上游异常翻译成合适的 HTTP 状态码**，别让 openai 的异常裸奔成 500（类比 Spring 的 `@ControllerAdvice`）：

```python
import openai
from fastapi import HTTPException

@app.post("/chat3")
async def chat3(req: ChatRequest,
                client: AsyncOpenAI = Depends(get_llm_client)) -> dict:
    try:
        resp = await client.chat.completions.create(
            model=os.environ.get("LLM_MODEL", "qwen3.5:27b"),
            messages=[{"role": "user", "content": req.message}],
        )
    except openai.RateLimitError:
        raise HTTPException(status_code=429, detail="上游限流，请稍后重试")
    except openai.APITimeoutError:
        raise HTTPException(status_code=504, detail="模型响应超时")
    except openai.APIStatusError as e:
        raise HTTPException(status_code=502, detail=f"上游异常: {e.status_code}")
    return {"reply": resp.choices[0].message.content}
```

**异步并发调用 LLM 的三个坑**（前置知识：[[async异步编程]]）：

1. **必须用 `AsyncOpenAI`**。在 `async def` 路由里调同步 `OpenAI` 客户端，会把整个事件循环卡死几十秒——所有其他请求一起挂。这是 FastAPI + LLM 最经典的性能事故。真要用同步库，路由写成普通 `def`（FastAPI 会丢到线程池），或者 `await asyncio.to_thread(...)`
2. **并发要限流**。`asyncio.gather` 一次开 100 个请求，云端直接返回一片 429。用 `asyncio.Semaphore` 压住并发数：

```python
import asyncio, os

MODEL = os.environ.get("LLM_MODEL", "qwen3.5:27b")
sem = asyncio.Semaphore(5)          # 同时最多 5 个在飞

async def ask(client, text: str) -> str:
    async with sem:
        resp = await client.chat.completions.create(
            model=MODEL, messages=[{"role": "user", "content": text}])
        return resp.choices[0].message.content

results = await asyncio.gather(*[ask(client, t) for t in texts],
                              return_exceptions=True)   # 一个失败不炸全场
```

3. **`gather` 默认一个失败全抛**。加 `return_exceptions=True` 让失败的那条以异常对象的形式返回，其余结果照常拿到，再单独重试失败项。

本地 Ollama 还有个额外限制：它**默认串行处理请求**，并发发 10 个不会真的并行，只是排队。想并行要调 `OLLAMA_NUM_PARALLEL`，但显存也会跟着涨。

### 6. 项目组织建议

```text
E:\workspace\AiStudy\phase1-llm-api\chat_service\
├── main.py          # app 创建、路由挂载
├── schemas.py       # pydantic 模型（DTO 层）
├── llm.py           # LLM 客户端与调用封装（Service 层）
└── .env             # LLM_BASE_URL / LLM_API_KEY / LLM_MODEL
```

## 动手任务

1. 在 `E:\workspace\AiStudy\phase1-llm-api\chat_service\` 按上面结构搭起服务，跑通 `/health` 与 `/chat2`（连本地 Ollama），并在 `/docs` 里试调
2. 实现 `/chat/stream` SSE 接口，用 curl -N 验证逐字输出；再故意把路由改成同步 `def` + 同步 client，用两个终端并发请求感受阻塞差异
3. 进阶：给 `/chat2` 增加 `session_id` 参数，用内存 dict 维护多会话历史（复用 [[Chat Completions API]] 的多轮逻辑），实现服务端有状态会话

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[流式输出与SSE]]
- [[Chat Completions API]]
- [[dataclass与pydantic]]
- [[类型注解与typing]]
- [[async异步编程]]
- [[Ollama本地模型]]
- [[AI学习地图]]
