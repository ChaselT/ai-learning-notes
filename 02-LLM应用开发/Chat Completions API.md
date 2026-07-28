---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Chat Completions API

## 这是什么/为什么重要

Chat Completions 是 OpenAI 定义的对话接口格式，如今已成为**事实标准**——DeepSeek、Qwen、Ollama、vLLM 全都提供兼容端点。学会这一个接口，等于拿到了整个 LLM 生态的钥匙，类比 JDBC：换数据库只换 URL 和驱动，代码不变。

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
# pip install openai
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],        # 从环境变量读，绝不硬编码
    base_url="https://api.deepseek.com/v1",        # 换 base_url 即换供应商
)

resp = client.chat.completions.create(
    model="deepseek-chat",
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
    resp = client.chat.completions.create(model="deepseek-chat", messages=history)
    answer = resp.choices[0].message.content
    history.append({"role": "assistant", "content": answer})  # 回填，下轮才"记得"
    return answer

print(chat("我叫老唐，是Java工程师"))
print(chat("我叫什么名字？"))   # 能答对，因为历史里有
```

漏掉回填 assistant 消息是新手第一大坑——表现为模型"完全不记得上一轮说过什么"。历史越长费用越高，需要截断/摘要策略（见 [[Token与上下文窗口]]）。

### 4. 主流模型与兼容端点

| 供应商 | base_url | 常用模型 | 说明 |
|---|---|---|---|
| OpenAI | `https://api.openai.com/v1` | gpt-4o, gpt-4o-mini | 原生标准 |
| DeepSeek | `https://api.deepseek.com/v1` | deepseek-chat, deepseek-reasoner | 便宜量大，国内可用 |
| Qwen(阿里) | `https://dashscope.aliyuncs.com/compatible-mode/v1` | qwen-plus, qwen-max | 兼容模式端点 |
| Anthropic | 官方是自家 Messages API | claude 系列 | 另有兼容层/网关可转 |
| Ollama 本地 | `http://localhost:11434/v1` | qwen2.5:14b | 零成本开发，见 [[Ollama本地模型]] |

同一份代码，改 `base_url` + `api_key` + `model` 三个值即可切换供应商——开发期用本地 Ollama，上线切云端。

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

## 动手任务

1. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_first_call.py`：完成第一次 API 调用，打印回复和 usage，密钥走环境变量
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_multi_turn.py`：实现终端多轮对话循环（`while True` + input），验证"报名字后下一轮还记得"；再故意注释掉回填 assistant 的那行，观察失忆现象
3. 写 `E:\workspace\AiStudy\phase1-llm-api\ex03_switch_backend.py`：用环境变量 `LLM_BASE_URL`/`LLM_MODEL` 控制，同一份代码分别跑通 DeepSeek 云端和本地 Ollama

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[Token与上下文窗口]]
- [[采样参数详解]]
- [[流式输出与SSE]]
- [[Ollama本地模型]]
- [[AI学习地图]]
