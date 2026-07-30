---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Token与上下文窗口

## 这是什么/为什么重要

Token 是 LLM 的最小处理单位，类比 JVM 的字节码指令：你写的是字符串，模型吃的是 token 序列。API 按 token 计费、上下文窗口按 token 限长、生成速度按 token/s 衡量——不理解 token，你既算不清成本，也解释不了"为什么对话久了模型忘事"。

## 核心内容

### 1. Tokenizer：字符串 → token 序列

模型不认识字符，输入前先由 tokenizer 切分并映射为整数 ID。切分粒度介于"字符"和"单词"之间（BPE 算法，常见词一个 token，生僻词拆成几段）。

```text
"Hello world"        → ["Hello", " world"]           → 2 tokens
"internationalization" → ["international", "ization"] → 2 tokens
"你好，世界"          → ["你好", "，", "世", "界"]     → 约 4 tokens
```

**中文吃 token**：英文平均 1 token ≈ 4 个字符（约 0.75 个单词），而中文通常 1 个汉字 ≈ 1~2 token。同样一段话，中文的 token 数往往是英文的 1.5~2 倍——意味着更高的费用和更快耗尽窗口。粗略换算记住：**1000 token ≈ 750 英文单词 ≈ 500~700 汉字**。

### 2. 上下文窗口（context window）

模型一次调用能处理的 token 总数上限 = **输入（全部历史消息）+ 输出**。常见规格：

| 模型 | 窗口大小 |
|---|---|
| gpt-5.6 系列（Sol/Terra/Luna） | 1M |
| claude-sonnet-5 / claude-opus-5 | 1M |
| deepseek-v4-flash / v4-pro | 128K 级 |
| Qwen3.5-27B（本地） | 256K（Ollama 默认只开 4K 左右，需调 `num_ctx`） |

2026 年主流窗口已经普遍拉到 128K~1M。但窗口大**不等于**该往里塞：输入是要花钱的，而且下一条"lost in the middle"的问题依然存在。

超窗口的表现：要么 API 直接报错，要么框架静默截断最早的消息——后者更危险，表现为"模型突然忘了开头说的话"。这是多轮对话应用最常见的线上问题。

另外注意"标称窗口"不等于"有效记忆"：很多模型在超长上下文的中段信息召回明显变差（lost in the middle），关键指令放开头（system）或结尾更稳。

### 3. 计费按 token

云端 API 按 `输入token数 × 输入单价 + 输出token数 × 输出单价` 计费，输出通常比输入贵 2~4 倍。多轮对话每次都要重发全部历史（见 [[Chat Completions API]]），所以**对话越长，每一轮的输入费用越高**，成本是 O(n²) 增长的。这也是为什么要做上下文管理。

### 4. 上下文管理策略

- **滑动窗口截断**：只保留最近 N 轮消息 + system prompt。实现最简单，缺点是丢失早期信息
- **摘要压缩**：历史超阈值时，调用一次 LLM 把旧消息压缩成一段摘要，作为 system 或首条消息保留。省 token 但摘要有信息损失
- **混合策略**（生产常用）：system prompt 永远保留 + 旧消息摘要 + 最近几轮原文

```python
def trim_history(messages: list[dict], max_messages: int = 20) -> list[dict]:
    """滑动窗口：保留 system + 最近 max_messages 条"""
    system = [m for m in messages if m["role"] == "system"]
    rest = [m for m in messages if m["role"] != "system"]
    return system + rest[-max_messages:]
```

### 5. 用 tiktoken 数 token

`uv add tiktoken`（OpenAI 官方 tokenizer 库，其他家模型分词器不同但数量级一致，估算够用）：

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")   # GPT-4o 之后（含 gpt-5.x）使用的编码
# 老模型（GPT-3.5/GPT-4）是 cl100k_base；不确定时用 encoding_for_model 让它自己选

for text in ["Hello world", "internationalization", "你好，世界", "面向对象编程"]:
    tokens = enc.encode(text)
    print(f"{text!r:30} -> {len(tokens)} tokens: {tokens}")

# 估算一次请求的输入成本（按 deepseek-v4-flash 输入 $0.14/百万 token）
history = "…拼接全部对话历史…"
n = len(enc.encode(history))
print(f"输入约 {n} tokens, 约 ${n / 1_000_000 * 0.14:.6f}")
```

注意 `tiktoken.encoding_for_model("gpt-5.6-terra")` 对**刚发布的新模型名**可能抛 `KeyError`（tiktoken 的模型映射表更新滞后于 API）。稳妥写法是直接 `get_encoding("o200k_base")`，或者 try/except 兜底。

非 OpenAI 模型想精确计数，用各家的 tokenizer（如 HuggingFace `AutoTokenizer.from_pretrained("Qwen/Qwen3.5-27B")`），或直接看 API 响应里的 `usage` 字段（最准，因为是服务端实际计的数）：

```python
resp = client.chat.completions.create(...)
print(resp.usage)  # prompt_tokens / completion_tokens / total_tokens
```

生产系统应把 `usage` 记进日志/监控：按用户、按功能聚合 token 消耗，是成本核算和异常检测（某用户疯狂刷接口）的基础数据。

省 token 实用技巧：

- system prompt 精炼再精炼，它每一轮都要重发
- few-shot 示例挑最短的有效版本，别贴长文档原文
- 让模型"只输出结果不解释"（输出更贵）
- 参考资料先做检索筛选，只塞相关段落而非整篇文档（RAG 的动机之一）
- 供应商的 prompt caching（前缀缓存）能对重复前缀打折，长 system prompt 场景收益明显。DeepSeek 是**自动命中**的：缓存命中的输入 token 只要 $0.0028/百万（未命中 $0.14），差 50 倍。命中条件是**前缀逐字节相同**——所以别在 system prompt 里插当前时间戳、随机 ID，也别每轮重排 messages 顺序，否则缓存永远命不中

### 6. Java 工程师视角的类比总结

| LLM 概念 | 类比 | 备注 |
|---|---|---|
| token | 字节码指令 | 用户看字符，运行时看 token |
| tokenizer | 编码器（如 UTF-8） | 不同模型的 tokenizer 不通用 |
| 上下文窗口 | 线程栈大小 -Xss | 超了就"溢出"，需主动管理 |
| usage 计费 | 云服务按量付费 | 输出比输入贵，历史重发导致成本随轮次上升 |
| 截断/摘要策略 | LRU 缓存淘汰 | 保新弃旧，重要信息（system）常驻

## 动手任务

1. 写 `E:\workspace\AiStudy\phase1-llm-api\ex02_token_count.py`：用 tiktoken 对比同一段意思的中英文文本（各约 200 字/词）的 token 数，验证"中文吃 token"；顺带试试代码片段、emoji、生僻字的 token 数，建立直觉
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex02_usage_watch.py`：连续进行 5 轮对话，每轮打印 `resp.usage.prompt_tokens`，观察输入 token 随轮次线性增长
3. 在上一题基础上实现 `trim_history`（保留 system + 最近 6 条），确认 prompt_tokens 不再无限增长

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[大语言模型工作原理速览]]
- [[Chat Completions API]]
- [[结构化输出]]
- [[Ollama本地模型]]
- [[流式输出与SSE]]
- [[AI学习地图]]
