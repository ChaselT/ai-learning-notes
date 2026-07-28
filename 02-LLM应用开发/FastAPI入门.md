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

```python
# pip install "fastapi[standard]" openai
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
        model=os.environ.get("LLM_MODEL", "qwen2.5:14b"),
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
            model=os.environ.get("LLM_MODEL", "qwen2.5:14b"),
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

### 5. 项目组织建议

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
