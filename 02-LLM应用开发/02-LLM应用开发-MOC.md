---
tags: [AI学习, 阶段1, LLM, MOC]
created: 2026-07-24
---

# 02-LLM应用开发 MOC

> 本阶段目标：从"会调 HTTP 接口的 Java 工程师"变成"能独立开发 LLM 应用的 AI 工程师"。
> 所有代码写到 `E:\workspace\AiStudy\phase1-llm-api\` 下。
>
> 本地硬件基线：RTX 2080 Ti 22GB 显存 + 64GB 内存，可流畅本地跑 Qwen2.5-14B（Q4）。

## 学习顺序

按以下顺序学习，前后有依赖关系，不建议跳跃：

| 序号 | 笔记 | 一句话简介 |
|---|---|---|
| 1 | [[大语言模型工作原理速览]] | 应用工程师视角理解 LLM：next token prediction、幻觉根源、模型命名怎么读 |
| 2 | [[Token与上下文窗口]] | Token 是 LLM 世界的"字节"：计费单位、窗口限制、上下文管理策略 |
| 3 | [[Chat Completions API]] | 与 LLM 对话的标准接口：消息角色、多轮对话本质、OpenAI 兼容生态 |
| 4 | [[采样参数详解]] | temperature/top_p 等参数怎么调：代码生成用低温，创意写作用高温 |
| 5 | [[流式输出与SSE]] | 为什么 ChatGPT 是一个字一个字蹦出来的：SSE 协议与 stream=True |
| 6 | [[Prompt工程]] | 把 prompt 当代码写：system prompt、few-shot、CoT、可套用模板 |
| 7 | [[结构化输出]] | 让 LLM 输出可被程序消费的 JSON：response_format + pydantic 校验 |
| 8 | [[Function Calling]] | 让 LLM 调用你的函数：工具调用完整闭环，Agent 的基石 |
| 9 | [[多模态视觉API]] | 给模型传图片：OCR、截图分析、图表理解，本地 Qwen2.5-VL 可行性 |
| 10 | [[Ollama本地模型]] | 22G 显存跑 qwen2.5:14b：本地开发零成本，OpenAI 兼容 API |
| 11 | [[FastAPI入门]] | Python 版 Spring Boot：把你的 LLM 能力包装成 HTTP 服务 |

## 项目①：CLI 智能助手

学完本阶段后完成，代码放 `E:\workspace\AiStudy\phase1-llm-api\project_cli_assistant\`。

**功能要求：**

1. **多轮对话**：维护对话历史，支持连续追问（上下文不丢失），支持 `/clear` 清空历史
2. **流式输出**：回复逐字打印到终端，不是等全部生成完再输出
3. **至少 3 个工具调用**（Function Calling），例如：
   - `get_current_time`：查询当前时间
   - `calculator`：数学计算（安全求值，不要裸 `eval`）
   - `read_file` / `search_web`：读本地文件或联网搜索（任选其一）
4. **可切换后端**：通过环境变量在 DeepSeek/Qwen 云端 API 与本地 Ollama 之间切换

**加分项：** 上下文超长时自动摘要压缩；工具并行调用；对话历史持久化到 JSON 文件。

**建议目录结构：**

```text
E:\workspace\AiStudy\phase1-llm-api\
├── .venv\                      # 虚拟环境
├── .env                        # 密钥与后端配置（进 .gitignore）
├── ex01_*.py ... ex10_*.py     # 各篇笔记的动手任务
├── prompts\                    # prompt 模板文件
├── chat_service\               # FastAPI 服务（见 [[FastAPI入门]]）
└── project_cli_assistant\      # 项目①
    ├── main.py                 # CLI 入口与对话循环
    ├── llm.py                  # 客户端封装、流式处理
    ├── tools.py                # 工具定义与注册表
    └── history.py              # 历史管理（截断/摘要/持久化）
```

## 环境准备清单

开始学习前把环境搭好，避免中途反复折腾：

- [ ] 用 uv 初始化项目（与阶段 0 一致，见 [[uv与项目管理]]）：在 `phase1-llm-api` 下 `uv init .`
- [ ] 基础依赖：`uv add openai tiktoken pydantic python-dotenv "fastapi[standard]" "httpx[socks]"`（本机有 SOCKS 代理，httpx 需 socks 支持）
- [ ] 注册 DeepSeek 开放平台（充 10 元够用整个阶段），API key 存入环境变量 `DEEPSEEK_API_KEY`
- [ ] 安装 Ollama，设置 `OLLAMA_MODELS` 到大容量盘，拉取 `qwen2.5:14b`（详见 [[Ollama本地模型]]）
- [ ] 建好项目目录 `E:\workspace\AiStudy\phase1-llm-api\`，初始化 git，`.env` 加入 `.gitignore`

## 学习节奏建议

在职学习，按每周 5~8 小时估算：

| 周次 | 内容 | 产出 |
|---|---|---|
| 第 1 周 | 原理速览 + Token + Chat API | ex01~ex03 跑通，云端与本地都调通 |
| 第 2 周 | 采样参数 + 流式 + Prompt 工程 | ex04~ex06，积累自己的 prompt 模板库 |
| 第 3 周 | 结构化输出 + Function Calling | ex07~ex08，工具调用闭环跑通 |
| 第 4 周 | 多模态 + Ollama 深入 + FastAPI | ex09~ex10 + chat_service 服务 |
| 第 5 周 | 项目①：CLI 智能助手 | 完整项目 + 自测通过验收标准 |

原则：**每篇笔记的动手任务必须做完再进下一篇**——LLM 开发的知识密度不高，但手感必须靠写代码建立，光看笔记会产生"我会了"的错觉。

## 常见问题

- **Q: 没有 OpenAI 账号影响学习吗？** 不影响。全程可用 DeepSeek（便宜）+ 本地 Ollama（免费），接口完全兼容，笔记示例改 base_url 即可
- **Q: 需要先学深度学习/PyTorch 吗？** 本阶段不需要。应用层开发只依赖 API 调用能力，原理层面 [[大语言模型工作原理速览]] 的深度已经够用
- **Q: Java 背景最容易踩的坑？** 忘记 API 是无状态的（历史要自己带）、在 async 路由里写阻塞调用、拿字符串拼 JSON 而不用 pydantic

## 阶段验收标准

能独立做到以下各项，即视为通过本阶段：

- [ ] 能口头解释：LLM 为什么会幻觉、多轮对话为什么要每次带全部历史
- [ ] 能不看笔记写出一个带流式输出的多轮对话脚本
- [ ] 能定义 tools schema 并跑通"模型请求调用 → 执行函数 → 回传结果"完整闭环
- [ ] 能用 pydantic 校验 LLM 的 JSON 输出，并实现失败重试
- [ ] 本地 Ollama 跑通 qwen2.5:14b，并用 openai SDK 连接它
- [ ] 用 FastAPI 写出一个 SSE 流式聊天接口，能用 curl 验证
- [ ] 项目①完成并通过自测
- [ ] **通过 [[阶段1测试]]（笔试+实操，Claude 批改 ≥80 分）——这是进入阶段 2 的硬门槛**

验收方式建议：把项目①的 README 和一段运行录屏（或终端输出截图）留档；对照上面清单逐项自查，卡住的项回到对应笔记重做动手任务。

完成本阶段后进入下一阶段：RAG 与知识库应用（笔记待建），Agent 相关内容见 [[04-Agent开发]] 占位。

## 相关笔记

- [[AI学习地图]]
- [[大语言模型工作原理速览]]
- [[Token与上下文窗口]]
- [[Chat Completions API]]
- [[采样参数详解]]
- [[流式输出与SSE]]
- [[Prompt工程]]
- [[结构化输出]]
- [[Function Calling]]
- [[多模态视觉API]]
- [[Ollama本地模型]]
- [[FastAPI入门]]
- 前置（阶段0）：[[async异步编程]]、[[dataclass与pydantic]]、[[类型注解与typing]]
- 后续（阶段2+）：[[04-Agent开发]]
- 本地环境速查：[[Ollama本地模型]]
