---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Chat Completions API

## 这是什么/为什么重要

Chat Completions 是 OpenAI 定义的对话接口格式，如今已成为**事实标准**——DeepSeek、Qwen、Ollama、vLLM 全都提供兼容端点。学会这一个接口，等于拿到了整个 LLM 生态的钥匙，类比 JDBC：换数据库只换 URL 和驱动，代码不变。

> OpenAI 自家现在主推更新的 **Responses API**（内置会话状态、原生工具调用），新项目如果**只用 OpenAI**可以直接学它。但 Chat Completions 并未废弃（OpenAI 官方 deprecations 页面至今没有给它排下线日期），而且它是**唯一被全行业实现的跨厂商接口**——DeepSeek、通义、Ollama、vLLM 都只提供它。本阶段以 Chat Completions 为基线，学完再看 Responses API 只是增量。

## 核心内容

### 1. 消息角色：system / user / assistant

请求体的核心是 `messages` 数组，每条消息有 `role` 和 `content`：

- **system**：给模型的"岗位说明书"，设定身份、规则、输出格式。用户看不到，优先级最高
- **user**：用户的输入
- **assistant**：模型的历史回复（多轮对话时由你回填）

```python
messages = [
    {"role": "system", "content": "你是资深Java专家，回答简洁，代码带注释。"},
    {"role": "user", "content": "HashMap 和 ConcurrentHashMap 的区别？"},
]
```

### 2. 最小可运行示例（openai SDK）

```python
# uv add openai "httpx[socks]"   # 本机走 SOCKS 代理，httpx 必须带 socks extra
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],        # 从环境变量读，绝不硬编码
    base_url="https://api.deepseek.com",           # 换 base_url 即换供应商
)

resp = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=[
        {"role": "system", "content": "你是一个乐于助人的助手。"},
        {"role": "user", "content": "用一句话解释什么是幂等性。"},
    ],
)
print(resp.choices[0].message.content)
print(resp.usage)  # token 用量，详见 [[Token与上下文窗口]]
```

### 3. 多轮对话的本质：每次带全部历史

**API 是无状态的，类比 HTTP**：服务端不保存任何会话，"上下文记忆"完全靠客户端每次把完整历史重发一遍。就像 HTTP 靠 Cookie/Session 机制自己维护状态，LLM 对话靠 `messages` 数组维护状态。

```python
history = [{"role": "system", "content": "你是一个助手。"}]

def chat(user_input: str) -> str:
    history.append({"role": "user", "content": user_input})
    resp = client.chat.completions.create(model="deepseek-v4-flash", messages=history)
    answer = resp.choices[0].message.content
    history.append({"role": "assistant", "content": answer})  # 回填，下轮才"记得"
    return answer

print(chat("我叫老唐，是Java工程师"))
print(chat("我叫什么名字？"))   # 能答对，因为历史里有
```

漏掉回填 assistant 消息是新手第一大坑——表现为模型"完全不记得上一轮说过什么"。历史越长费用越高，需要截断/摘要策略（见 [[Token与上下文窗口]]）。

### 4. 主流模型与兼容端点

| 供应商 | base_url | 常用模型（2026-07） | 说明 |
|---|---|---|---|
| OpenAI | `https://api.openai.com/v1` | gpt-5.6-terra, gpt-5.6-luna | 原生标准；官方新项目主推 Responses API |
| DeepSeek | `https://api.deepseek.com` | deepseek-v4-flash, deepseek-v4-pro | 便宜量大，国内可用，本阶段主力 |
| Qwen(阿里百炼) | `https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com/compatible-mode/v1` | qwen3.7-flash, qwen3.7-plus | 兼容模式端点，WorkspaceId 在控制台取 |
| Anthropic | 官方是自家 Messages API | claude-sonnet-5, claude-opus-5 | 另有兼容层/网关可转 |
| Ollama 本地 | `http://localhost:11434/v1` | qwen3.5:27b | 零成本开发，见 [[Ollama本地模型]] |

同一份代码，改 `base_url` + `api_key` + `model` 三个值即可切换供应商——开发期用本地 Ollama，上线切云端。

模型名迭代很快（DeepSeek 在 2026-07 就把 `deepseek-chat` / `deepseek-reasoner` 换成了 `deepseek-v4-flash` / `deepseek-v4-pro`）。**别把模型名写死在代码里**，一律走环境变量 `LLM_MODEL`，换代时改 `.env` 不改代码。

### 4.1 成本速查（2026-07，美元/百万 token）

| 模型 | 输入（未命中缓存） | 输入（命中缓存） | 输出 |
|---|---|---|---|
| deepseek-v4-flash | $0.14 | $0.0028 | $0.28 |
| deepseek-v4-pro | $0.435 | $0.003625 | $0.87 |
| gpt-5.6-luna | $1.00 | — | $6.00 |
| claude-sonnet-5 | $3.00 | — | $15.00 |
| 本地 Ollama | $0 | $0 | $0（只费电） |

学习阶段用 deepseek-v4-flash：充 10 元人民币，够跑完整个阶段 1 还有富余。**缓存命中价便宜 50 倍**——多轮对话里前缀（system + 历史）不变时会自动命中，这也是"别每轮重排 messages 顺序"的经济学理由。

### 5. API key 管理

- 密钥放**环境变量**或 `.env` 文件（配合 `python-dotenv`），`.env` 必须进 `.gitignore`
- 绝不硬编码进源码、不提交到 git、不贴到聊天工具里
- 泄漏等于泄漏你的账单：被扫到就会被人刷爆额度

```python
# .env 文件内容:  DEEPSEEK_API_KEY=sk-xxxx
# pip install python-dotenv
from dotenv import load_dotenv
load_dotenv()  # 之后 os.environ["DEEPSEEK_API_KEY"] 即可读到
```

PowerShell 设置持久环境变量：`setx DEEPSEEK_API_KEY "sk-xxxx"`（重开终端生效）。

团队/生产环境的进阶做法：密钥放配置中心或 Secret 管理服务（类比 Spring Cloud Config + Vault），并为不同环境使用不同 key，便于按环境限额与审计。

### 6. 响应对象速查

`resp.choices[0]` 下最常用的三个字段：

- `.message.content`：回复文本（工具调用时可能为 None，见 [[Function Calling]]）
- `.message.role`：恒为 `"assistant"`
- `.finish_reason`：`"stop"` 正常结束 / `"length"` 被 max_tokens 截断 / `"tool_calls"` 请求调用工具——生产代码应当检查它（见 [[采样参数详解]]）

### 7. 错误处理与重试（生产必备）

LLM API 是**必然会失败**的远程调用，比普通 HTTP 接口更不稳定：限流（429）、上游过载（503/529）、生成慢导致超时都是常态。Java 里你会给 Feign 加 Resilience4j 重试，这里同理。

openai SDK **自带重试**：默认对连接错误、408/409/429/5xx 做 2 次指数退避重试，超时默认 10 分钟。多数场景配好这两个参数就够了：

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
    timeout=60.0,      # 秒；流式长回答可调大，非流式建议 30~120
    max_retries=3,     # 0 表示关闭内置重试
)

# 单次调用临时覆盖，不影响 client
resp = client.with_options(timeout=10.0).chat.completions.create(...)
```

需要自己处理时，**按异常类型分支**，不要 `except Exception` 一把抓——可重试和不可重试必须分开：

```python
import openai

try:
    resp = client.chat.completions.create(model=MODEL, messages=messages)
except openai.RateLimitError as e:            # 429：限流，等一下再来
    retry_after = e.response.headers.get("retry-after", "5")
    print(f"限流，{retry_after}s 后重试")
except openai.APITimeoutError:                # 超时：可重试
    print("超时，重试或降级到小模型")
except openai.APIStatusError as e:            # 其他非 2xx
    if e.status_code >= 500:
        print(f"服务端错误 {e.status_code}，可重试")
    else:
        print(f"请求有问题（{e.status_code}），重试也没用：{e.message}")
except openai.APIConnectionError:             # 网络层失败（代理挂了常见）
    print("连不上，检查网络/代理")
```

常见异常类速查：`BadRequestError`(400) / `AuthenticationError`(401) / `PermissionDeniedError`(403) / `NotFoundError`(404，模型名写错最常见) / `RateLimitError`(429) / `APIStatusError`(其他状态码基类) / `APITimeoutError` / `APIConnectionError`。

工程要点：

- **4xx 不要重试**（除 429）——参数错、模型名错、key 错，重试一百次还是错，只会白烧时间
- 重试要有**上限 + 指数退避 + 抖动**，否则限流时全体客户端同时重试会把上游打得更死
- 重试是**幂等假设**：LLM 调用本身无副作用可以重试，但如果这次调用触发了工具（下单、发邮件），重试前要想清楚（见 [[Function Calling]]）
- 给用户面向的路径准备**降级方案**：大模型失败 → 小模型 → 本地 Ollama → 友好报错，别直接抛 500

### 8. 本机网络：httpx 必须带 socks

本机系统装了 SOCKS 代理，而 openai SDK 底层用 httpx 发请求。默认安装的 httpx **不支持 socks5 协议**，表现为 `APIConnectionError` 或直接卡死。装依赖时务必带上 extra：

```powershell
uv add openai "httpx[socks]"
```

验证代理是否被识别：httpx 会读 `HTTPS_PROXY`/`ALL_PROXY` 环境变量。如果想让某个 client 显式走代理：

```python
from openai import OpenAI, DefaultHttpxClient

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
    http_client=DefaultHttpxClient(proxy="socks5://127.0.0.1:10808"),
)
```

用 `DefaultHttpxClient` 而不是裸 `httpx.Client`，才能保留 SDK 默认的超时和连接池配置。**连本地 Ollama 时要绕过代理**（`NO_PROXY=localhost,127.0.0.1`），否则本机请求被送去代理会直接失败。

## 动手任务

1. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_first_call.py`：完成第一次 API 调用，打印回复和 usage，密钥走环境变量；给 client 配上 `timeout`/`max_retries`，并把模型名故意写错一次，捕获 `NotFoundError` 打印友好提示（体会"4xx 不该重试"）
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_multi_turn.py`：实现终端多轮对话循环（`while True` + input），验证"报名字后下一轮还记得"；再故意注释掉回填 assistant 的那行，观察失忆现象
3. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_switch_backend.py`：用环境变量 `LLM_BASE_URL`/`LLM_MODEL` 控制，同一份代码分别跑通 DeepSeek 云端和本地 Ollama

## 相关笔记

> **本阶段第 3/11 课** | 上一篇：[[Token与上下文窗口]] | **下一篇：[[采样参数详解]]**
> 学习顺序总表见 [[02-LLM应用开发-MOC]]

- [[02-LLM应用开发-MOC]]
- [[Token与上下文窗口]]
- [[采样参数详解]]
- [[流式输出与SSE]]
- [[Ollama本地模型]]
- [[AI学习地图]]
