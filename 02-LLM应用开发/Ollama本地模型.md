---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Ollama本地模型

## 这是什么/为什么重要

Ollama 是本地跑开源 LLM 的"Docker"：一条命令拉模型、一条命令跑起来，自动量化、自动管理显存，还内置 OpenAI 兼容 API。对你意味着：**开发调试零 API 成本、数据不出本机、断网可用**。你的 RTX 2080 Ti 魔改 22GB 是本地跑 14B 模型的理想配置，务必用起来。

## 核心内容

### 1. 安装（Windows）

从 https://ollama.com/download 下载 `OllamaSetup.exe` 安装，装完后台常驻（托盘图标），服务监听 `http://localhost:11434`。验证：

```powershell
ollama --version
curl http://localhost:11434        # 返回 "Ollama is running"
```

### 2. OLLAMA_MODELS：改模型存储路径

模型默认存到 `C:\Users\<用户名>\.ollama\models`，14B/32B 动辄十几 GB，强烈建议移到大盘。设置系统环境变量后**重启 Ollama**（托盘退出再启动）：

```powershell
# 以管理员运行，或在"系统属性-环境变量"图形界面设置
[Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "E:\ollama\models", "User")
```

其他常用环境变量：`OLLAMA_KEEP_ALIVE=30m`（模型驻留显存时长，默认 5m，频繁调用建议调大）、`OLLAMA_HOST=0.0.0.0`（允许局域网访问）。

### 3. 基本命令

```powershell
ollama pull qwen2.5:14b     # 拉取模型（默认 Q4_K_M 量化）
ollama run qwen2.5:14b      # 命令行交互式对话（/bye 退出）
ollama list                 # 已下载的模型
ollama ps                   # 正在运行的模型 + 实际显存占用
ollama rm qwen2.5:7b        # 删除模型
ollama show qwen2.5:14b     # 查看模型信息（参数量/量化/上下文长度）
```

### 4. 推荐型号（针对你的 22G 显存）

| 模型 | 用途 | 显存占用（约） |
|---|---|---|
| `qwen2.5:14b` | **日常主力**：对话、总结、function calling | 9~10 GB |
| `qwen2.5:32b-instruct-q4_K_M` | 尝鲜更强能力，速度较慢 | 19~20 GB（贴边） |
| `qwen2.5-coder:14b` | 代码生成/补全专用 | 9~10 GB |
| `qwen2.5vl:7b` | 看图（见 [[多模态视觉API]]） | 7~9 GB |

### 5. OpenAI 兼容 API：本地开发零成本

Ollama 暴露 `/v1` 兼容端点，openai SDK 直接连，**与云端代码完全一致**（见 [[Chat Completions API]]）：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",  # 必填但内容任意
)
resp = client.chat.completions.create(
    model="qwen2.5:14b",
    messages=[{"role": "user", "content": "用一句话解释什么是量化"}],
)
print(resp.choices[0].message.content)
```

流式、function calling、JSON mode 都支持（14B 的工具调用能力可用，复杂场景不如云端旗舰稳）。开发模式建议：**日常联调用本地 14B（免费、无限量），关键效果验证再切云端**，用环境变量一键切换（见 [[Chat Completions API]] 的动手任务 3）。

注意：Ollama 默认上下文窗口较小（多为 4096），长对话需要调大：

```python
# 方式1: 请求级 (Ollama 原生参数, 通过 extra_body 透传)
resp = client.chat.completions.create(
    model="qwen2.5:14b", messages=[...],
    extra_body={"options": {"num_ctx": 16384}},
)
# 方式2: 环境变量 OLLAMA_CONTEXT_LENGTH=16384 (新版本支持)
```

上下文越大，KV cache 吃的显存越多，22G 显存跑 14B 开 16K~32K 没问题。

### 6. 显存占用估算方法

```text
总显存 ≈ 权重 + KV cache + 少量运行时开销

权重 ≈ 参数量 × 每参数字节数
  FP16: 2 字节/参数    → 14B ≈ 28 GB（你跑不动，所以要量化）
  Q8:   ~1.06 字节     → 14B ≈ 15 GB
  Q4_K_M: ~0.55 字节   → 14B ≈ 8 GB，32B ≈ 18 GB

KV cache ≈ 随上下文长度线性增长，14B 开 8K 约 1~2 GB，32K 约 4~6 GB
```

实测永远优于估算：跑起来后 `ollama ps` 看真实占用，`nvidia-smi` 看整卡情况。如果显示 `xx%/xx% CPU/GPU`，说明显存不够被迫分层到内存，速度会断崖式下降——此时换小模型或降上下文。

## 动手任务

1. 安装 Ollama，设置 `OLLAMA_MODELS` 到非 C 盘，拉取 `qwen2.5:14b` 并用 `ollama run` 对话；用 `ollama ps` 记录显存占用，与估算公式对比
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex10_ollama_openai.py`：用 openai SDK 连本地 Ollama，跑通普通调用 + 流式输出
3. 压力测试：拉 `qwen2.5:32b-instruct-q4_K_M`，观察 `ollama ps` 与 `nvidia-smi`，记录 tokens/s 的体感差异（可对比 14b），结论写进脚本注释

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[大语言模型工作原理速览]]
- [[Chat Completions API]]
- [[Token与上下文窗口]]
- [[多模态视觉API]]
- [[AI学习地图]]
