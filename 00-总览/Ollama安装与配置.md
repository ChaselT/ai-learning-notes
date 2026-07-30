---
tags: [AI学习, 总览, 环境, Ollama]
created: 2026-07-29
---

# Ollama 安装与配置（本机专属）

> 这是**本机的实操手册**（含你机器上的历史遗留与坑），概念和用法见 [[Ollama本地模型]]。
> 本机现状（2026-07-29 检测）：**程序已卸载**，但模型目录 `F:\ollama_models` 还在（18.49 GB，内含一个 `qwq`），`OLLAMA_MODELS` 系统级环境变量仍指向它——**重装后会直接认出旧模型，不用重新配置**。

## 一、安装前确认

| 项 | 本机状态 | 说明 |
|----|----------|------|
| 显存 | RTX 2080 Ti 魔改 **22 GB** | 决定能跑多大模型，见下方选型表 |
| 模型盘 | `F:\ollama_models` | 环境变量已设（系统级），别再改 |
| F 盘空间 | 约 117 GB 可用 | 删 qwq(+18.5) 后拉 27b(-17)，绰绰有余 |
| 系统代理 | **有 SOCKS 代理** | ⚠️ 必须配 `NO_PROXY`，否则连不上本地服务 |

## 二、安装步骤

1. 到 https://ollama.com/download 下载 **Windows** 版安装包（`OllamaSetup.exe`），当前主线版本 v0.32.x（以官网为准）
2. 双击安装，一路默认即可——程序装到 `%LOCALAPPDATA%\Programs\Ollama`，**模型不会装在这里**（由 `OLLAMA_MODELS` 决定）
3. 安装完成后 Ollama 会作为后台服务启动，任务栏托盘出现羊驼图标
4. **新开一个 PowerShell**（重要：老窗口读不到新 PATH），验证：

```powershell
ollama --version
ollama list          # 应该能看到 qwq —— 证明旧模型目录被正确识别
```

如果 `ollama list` 是空的，说明 `OLLAMA_MODELS` 没生效，检查：

```powershell
[Environment]::GetEnvironmentVariable("OLLAMA_MODELS","Machine")   # 应为 F:\ollama_models
```

改动环境变量后必须**重启 Ollama 服务**（托盘图标右键退出，再从开始菜单启动）才生效。

## 三、必做配置

### 1. NO_PROXY（本机必配，否则连不上）

本机有 SOCKS 代理，Python 的 httpx/openai SDK 会读系统代理设置，把发往 `localhost:11434` 的请求也送去代理服务器，表现为**莫名其妙的连接失败**。

- 项目里：`phase1-llm-api\.env` 已写入 `NO_PROXY=localhost,127.0.0.1`
- 系统级（一劳永逸，推荐也配上）：

```powershell
[Environment]::SetEnvironmentVariable("NO_PROXY","localhost,127.0.0.1","User")
```

### 2. 其他可选环境变量（按需，都要重启服务）

| 变量 | 作用 | 建议值 |
|------|------|--------|
| `OLLAMA_KEEP_ALIVE` | 模型在显存里驻留多久后卸载 | `30m`（频繁调试时避免反复加载 17 GB） |
| `OLLAMA_HOST` | 监听地址 | 默认 `127.0.0.1:11434`；只有要从别的机器访问才改 |
| `OLLAMA_NUM_PARALLEL` | 并发处理请求数 | 默认较小，**并发调用会排队**——学 async 并发时留意这点 |

## 四、模型管理

### 清理旧模型

```powershell
ollama rm qwq        # 腾出 18.49 GB
```

> qwq 是 2025 年的 32B 推理模型，对阶段 1 不合适：占显存大、输出前会吐一大段思考过程、且**推理模型往往不支持 temperature**（`ex04` 的温度实验做不了）。

### 拉取主力模型

```powershell
ollama pull qwen3.5:27b        # 约 17 GB，256K 上下文，原生支持传图
```

### 22 GB 显存选型速查（2026-07）

| 模型 | 体积 | 说明 |
|------|------|------|
| **qwen3.5:27b** | ~17 GB | ⭐ 主力：能力强、原生多模态、上下文 256K |
| qwen3:30b（MoE） | ~19 GB | 速度快得多（只激活部分参数），适合要快的场景 |
| qwen3-coder:30b | ~19 GB | 写代码专用 |
| qwen3-vl:8b | ~6.1 GB | 轻量视觉模型，需 Ollama ≥ 0.12.7 |
| ~~qwen3.5:35b~~ | ~24 GB | ❌ **超出 22 GB，别碰** |

⚠️ **MoE 模型的显存要按总参数算，不是按激活参数算**——`-A3B` 这类命名里的 3B 是激活量，别拿它估显存。

⚠️ 别把显存吃满：模型权重之外还要留几 GB 给 **KV cache**（上下文越长吃得越多）。27b 占 17 GB，剩 5 GB 给上下文，日常够用；想开超长上下文就得量化 KV 或换小模型。

## 五、验证清单

```powershell
# 1. 服务活着
ollama list

# 2. 能对话
ollama run qwen3.5:27b
# >>> 你好，用一句话介绍你自己
# （Ctrl+D 退出）

# 3. 显存占用符合预期（模型加载后再看）
nvidia-smi

# 4. 正在跑哪些模型
ollama ps
```

**最关键的一步——OpenAI 兼容端点**（阶段 1 全靠它）：

```powershell
cd E:\workspace\AiStudy\phase1-llm-api
uv run python -c "from openai import OpenAI; c=OpenAI(base_url='http://localhost:11434/v1', api_key='ollama'); print(c.chat.completions.create(model='qwen3.5:27b', messages=[{'role':'user','content':'说三个字'}]).choices[0].message.content)"
```

这条命令通了，说明**你的 Python 代码可以像调云端 API 一样调本地模型**——后续所有练习换个 `base_url` 就能在本地/云端之间切。

## 六、故障排查

| 症状 | 大概率原因 | 处理 |
|------|-----------|------|
| Python 连不上 `localhost:11434` | **SOCKS 代理劫持**（本机头号坑） | 配 `NO_PROXY=localhost,127.0.0.1`，重开终端 |
| `ollama` 命令找不到 | 终端是安装前开的 | 新开 PowerShell |
| `ollama list` 是空的 | `OLLAMA_MODELS` 未生效 | 检查环境变量 + 重启 Ollama 服务 |
| 拉模型极慢/中断 | 网络 | 断点续传：重跑 `ollama pull` 即可 |
| 加载模型后爆显存 / 极慢 | 模型超出 22 GB，退化到 CPU 推理 | `nvidia-smi` 看占用，换小一号的模型 |
| 回答一半卡住 | 上下文超了 KV cache 空间 | 缩短对话历史，或换小模型 |
| 并发请求变串行 | `OLLAMA_NUM_PARALLEL` 默认值小 | 调大该变量；本地并发能力本就有限，别拿它压测 |

## 七、和学习的衔接

- 阶段 1：`ex10_ollama_openai.py` 专门练本地模型接入；日常调试用本地模型可以**零成本**试错，把云端额度留给最后验证
- 阶段 4：微调产出的模型可以量化成 GGUF 后用 `ollama create` 导入本机运行——那时会再回到这份文档

## 相关笔记

- [[Ollama本地模型]] —— 概念、参数与用法详解
- [[硬件与环境]] —— 本机配置与能力边界
- [[02-LLM应用开发-MOC]] —— 阶段 1 总览
