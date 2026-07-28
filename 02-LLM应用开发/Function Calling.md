---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Function Calling

## 这是什么/为什么重要

LLM 自己不会查天气、不会算 38 位乘法、不知道你数据库里有什么。Function Calling 让你把函数"注册"给模型：模型判断何时需要、以什么参数调用，**实际执行仍在你的代码里**，模型只负责决策和总结。类比 Java 的策略模式 + 反射调用：模型是路由器，你的函数是策略实现。这是 [[04-Agent开发]] 的基石——Agent 本质就是"循环执行 Function Calling"。

## 核心内容

### 1. 完整流程（五步闭环）

```text
① 你在请求里声明 tools schema（函数名/描述/参数JSON Schema）
② 模型判断需要工具 → 返回 tool_calls（函数名 + JSON 参数），而不是文本回答
③ 你的代码解析 tool_calls，真正执行本地函数
④ 把执行结果以 role="tool" 消息回传给模型
⑤ 模型综合工具结果，生成最终自然语言回答
```

关键认知：**模型从不执行代码**，它只生成"我想调 xxx(参数)"的意图；执行、鉴权、安全都是你的责任。

### 2. 完整可运行示例（查天气 + 计算器）

```python
import json, os
from openai import OpenAI

client = OpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                base_url="https://api.deepseek.com/v1")
MODEL = "deepseek-chat"

# --- 本地函数（真实项目里是查API/查库） ---
def get_weather(city: str) -> str:
    fake_db = {"北京": "晴 32℃", "上海": "小雨 28℃"}
    return fake_db.get(city, f"{city}: 暂无数据")

def calculator(expression: str) -> str:
    allowed = set("0123456789+-*/(). ")
    if not set(expression) <= allowed:      # 白名单过滤，绝不裸 eval 用户输入
        return "错误: 含非法字符"
    return str(eval(expression))

TOOLS = [
    {"type": "function", "function": {
        "name": "get_weather",
        "description": "查询指定城市的当前天气",
        "parameters": {"type": "object",
                       "properties": {"city": {"type": "string", "description": "城市名，如'北京'"}},
                       "required": ["city"]}}},
    {"type": "function", "function": {
        "name": "calculator",
        "description": "计算数学表达式，支持加减乘除和括号",
        "parameters": {"type": "object",
                       "properties": {"expression": {"type": "string", "description": "如 '3*(4+5)'"}},
                       "required": ["expression"]}}},
]
REGISTRY = {"get_weather": get_weather, "calculator": calculator}

def run(question: str) -> str:
    messages = [{"role": "user", "content": question}]
    while True:  # 循环：模型可能连续多轮要求调工具
        resp = client.chat.completions.create(model=MODEL, messages=messages,
                                              tools=TOOLS, temperature=0)
        msg = resp.choices[0].message
        if not msg.tool_calls:              # 不再调工具 -> 最终回答
            return msg.content
        messages.append(msg)                # 必须先回填模型的 tool_calls 消息
        for tc in msg.tool_calls:           # 可能一次返回多个（并行调用）
            args = json.loads(tc.function.arguments)
            result = REGISTRY[tc.function.name](**args)
            print(f"[调用] {tc.function.name}({args}) -> {result}")
            messages.append({"role": "tool",
                             "tool_call_id": tc.id,   # 必须带 id 与调用对应
                             "content": str(result)})

print(run("北京和上海现在天气怎么样？温度差几度？"))
```

### 3. 易错点清单

- 模型的 `tool_calls` 消息**必须原样 append 回 messages**，再跟 `role="tool"` 的结果消息，且 `tool_call_id` 一一对应，否则 API 报错
- `function.arguments` 是 **JSON 字符串**，要 `json.loads`；模型可能给出不合法参数，执行前应校验（可用 pydantic，见 [[结构化输出]]）
- `description` 写得越清楚，模型选工具、填参数越准——这是 prompt 工程的一部分（见 [[Prompt工程]]）
- 工具执行抛异常时，把错误信息作为 tool 结果回传（"查询失败：超时"），让模型向用户解释，别让整个请求 500
- 决策要稳定，temperature 用 0~0.3（见 [[采样参数详解]]）

### 4. 并行工具调用

模型一次可返回多个 `tool_calls`（如上例同时查两个城市）。上面代码是串行执行；I/O 型工具可用 `asyncio.gather` 并发执行以降低延迟（见 [[async异步编程]]）：

```python
import asyncio

async def exec_all(tool_calls) -> list[dict]:
    async def one(tc):
        args = json.loads(tc.function.arguments)
        result = await asyncio.to_thread(REGISTRY[tc.function.name], **args)
        return {"role": "tool", "tool_call_id": tc.id, "content": str(result)}
    return await asyncio.gather(*[one(tc) for tc in tool_calls])
```

不希望并行时，可传 `parallel_tool_calls=False` 强制一次只调一个（部分兼容端点支持）。另外给循环加**最大轮数上限**（如 10 轮），防止模型陷入"反复调工具不收敛"的死循环烧钱。

### 5. 这就是 Agent 的雏形

上面 `run()` 里的 `while True` 循环 = 最小 Agent：**感知（读对话）→ 决策（选工具）→ 行动（执行）→ 观察（结果回传）→ 循环**。后续阶段的 Agent 框架（LangGraph 等）只是给这个循环加上规划、记忆、多 Agent 协作，内核不变。详见 [[04-Agent开发]]。

## 动手任务

1. 把示例跑通：`E:\workspace\AiStudy\phase1-llm-api\ex08_function_calling.py`，观察"温度差几度"如何触发 天气→天气→计算器 的多轮调用链
2. 新增工具 `get_current_time()`（返回本机时间）和 `read_file(path)`（限定只能读 `E:\workspace\AiStudy` 目录内文件，做路径校验），扩展到 4 个工具：`E:\workspace\AiStudy\phase1-llm-api\ex08_more_tools.py`
3. 破坏性实验：把 `get_weather` 的 description 改成含糊的"一个函数"，观察模型还能不能正确选择工具；恢复后对比，体会 description 的重要性

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[Chat Completions API]]
- [[结构化输出]]
- [[Prompt工程]]
- [[采样参数详解]]
- [[async异步编程]]
- [[04-Agent开发]]
- [[AI学习地图]]
