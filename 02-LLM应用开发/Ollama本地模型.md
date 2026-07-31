---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# Ollama本地模型

## 这是什么/为什么重要

Ollama 是本地跑开源 LLM 的"Docker"：一条命令拉模型、一条命令跑起来，自动量化、自动管理显存，还内置 OpenAI 兼容 API。对你意味着：**开发调试零 API 成本、数据不出本机、断网可用**。你的 RTX 2080 Ti 魔改 22GB 显存，正好卡在能舒服跑 27B~30B 级模型的位置，务必用起来。

> 本篇按 Ollama **v0.32.x**（2026-07）写。Ollama 迭代很快，命令基本稳定，但模型库更新频繁——拉之前先去 https://ollama.com/library 看一眼当前有什么。

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
ollama pull qwen3.5:27b     # 拉取模型（默认 Q4_K_M 量化）
ollama run qwen3.5:27b      # 命令行交互式对话（/bye 退出）
ollama list                 # 已下载的模型
ollama ps                   # 正在运行的模型 + 实际显存占用
ollama rm qwen3:8b          # 删除模型
ollama show qwen3.5:27b     # 查看模型信息（参数量/量化/上下文长度）
```

### 4. 推荐型号（针对你的 22G 显存，2026-07 版）

| 模型 tag | 用途 | 下载体积 / 显存（约） |
|---|---|---|
| `qwen3.5:27b` | **日常主力**：对话、总结、function calling，且**原生支持传图**，256K 上下文 | 17 GB |
| `qwen3:30b` | MoE 架构（30B 总参 / 约 3B 激活），**推理速度明显快于同体积稠密模型** | 约 19 GB |
| `qwen3-coder:30b` | 代码生成/补全专用（MoE） | 约 19 GB |
| `qwen3-vl:8b` | 轻量看图，留足显存跑别的（见 [[多模态视觉API]]） | 6.1 GB |
| `qwen3-vl:30b` | 更强看图，贴边但能跑 | 20 GB |
| `gpt-oss:20b` | OpenAI 开放权重模型，换个"口味"做对比 | 约 14 GB |

选型建议：**先只拉 `qwen3.5:27b` 一个**。它文本能力够强、能看图、256K 上下文，本阶段所有练习（含多模态那篇）都能用它跑通，省得下一堆几十 GB 的模型占盘。等确实遇到瓶颈（代码补全不够好 / 想比速度）再补拉专用模型。

`qwen3.5:35b`（24 GB）超出你的 22G，会被迫分层到内存，速度断崖式下降——别碰。

> **MoE 是什么**：Mixture of Experts，模型有 30B 参数但每次前向只激活其中约 3B。显存要按**总参数**算（还是 19 GB），但速度按**激活参数**算（快得多）。2026 年开源模型大量转向 MoE，看到 `30b-a3b` 这种命名就是"30B 总参 / 3B 激活"。

### 5. OpenAI 兼容 API：本地开发零成本

Ollama 暴露 `/v1` 兼容端点，openai SDK 直接连，**与云端代码完全一致**（见 [[Chat Completions API]]）：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",  # 必填但内容任意
)
resp = client.chat.completions.create(
    model="qwen3.5:27b",
    messages=[{"role": "user", "content": "用一句话解释什么是量化"}],
)
print(resp.choices[0].message.content)
```

流式、function calling、JSON mode 都支持（27B 级的工具调用已经相当可用，复杂多轮工具链仍不如云端旗舰稳）。开发模式建议：**日常联调用本地模型（免费、无限量、不怕刷爆额度），关键效果验证再切云端**，用环境变量一键切换（见 [[Chat Completions API]] 的动手任务 3）。

⚠️ **连本地要绕过代理**：本机系统装了 SOCKS 代理，如果 `HTTPS_PROXY`/`ALL_PROXY` 生效，`http://localhost:11434` 的请求也会被送去代理，直接连不上。设 `NO_PROXY=localhost,127.0.0.1`，或在代码里给本地 client 显式关掉代理。

注意：Ollama 默认上下文窗口不大（通常 4096），模型标称的 256K 不会自动生效，长对话需要调大：

```python
# 方式1: 请求级 (Ollama 原生参数, 通过 extra_body 透传)
resp = client.chat.completions.create(
    model="qwen3.5:27b", messages=[...],
    extra_body={"options": {"num_ctx": 16384}},
)
# 方式2: 环境变量 OLLAMA_CONTEXT_LENGTH=16384 (服务级默认值，改完要重启 Ollama)
```

上下文越大，KV cache 吃的显存越多。22G 显存跑 27B(17GB 权重) 时，留给 KV cache 的只剩约 4~5 GB，开到 16K 比较稳妥，32K 要看实测。想开更大就换 `qwen3-vl:8b` 这类小模型，或开 `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` 把 KV cache 也量化掉。

### 6. 显存占用估算方法

```text
总显存 ≈ 权重 + KV cache + 少量运行时开销

权重 ≈ 参数量 × 每参数字节数
  FP16: 2 字节/参数    → 27B ≈ 54 GB（你跑不动，所以要量化）
  Q8:   ~1.06 字节     → 27B ≈ 29 GB（还是跑不动）
  Q4_K_M: ~0.55 字节   → 27B ≈ 15~17 GB，30B ≈ 19 GB，8B ≈ 5~6 GB

KV cache ≈ 随上下文长度线性增长，27B 开 8K 约 2~3 GB，32K 约 8 GB+
```

对照实际：`qwen3.5:27b` 官方标 17 GB，正好落在 Q4_K_M 的估算区间里——公式是准的。MoE 模型（如 `qwen3:30b`）**权重按总参数算**，别按激活参数算，否则会严重低估。

实测永远优于估算：跑起来后 `ollama ps` 看真实占用，`nvidia-smi` 看整卡情况。如果显示 `xx%/xx% CPU/GPU`，说明显存不够被迫分层到内存，速度会断崖式下降——此时换小模型或降上下文。

## 动手任务

1. 安装 Ollama，设置 `OLLAMA_MODELS` 到非 C 盘，拉取 `qwen3.5:27b` 并用 `ollama run` 对话；用 `ollama ps` 记录显存占用，与估算公式对比（官方标 17 GB，你实测多少？差值是什么？）
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex10_ollama_openai.py`：用 openai SDK 连本地 Ollama，跑通普通调用 + 流式输出；记得处理好代理绕过（`NO_PROXY`）
3. 对比实验：再拉一个小模型（如 `qwen3-vl:8b`，6.1 GB），同一个 prompt 分别跑 27b 和 8b，记录 tokens/s 与回答质量差异；再把 `num_ctx` 从默认调到 16384，观察 `ollama ps` 里显存的变化。结论写进脚本注释

## 相关笔记

> **本阶段第 10/11 课** | 上一篇：[[多模态视觉API]] | **下一篇：[[FastAPI入门]]**
> 学习顺序总表见 [[02-LLM应用开发-MOC]]

- [[02-LLM应用开发-MOC]]
- [[大语言模型工作原理速览]]
- [[Chat Completions API]]
- [[Token与上下文窗口]]
- [[多模态视觉API]]
- [[AI学习地图]]
