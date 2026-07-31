---
tags: [AI学习, 总览, 实战资产, 简历]
created: 2026-07-31
---

# 实战资产：video-clipper-pro

> 位置：`E:\workspace\python\video-clipper-pro` ｜ 商业软件（收费，不开源）｜ v0.4.1
> 2026-07-31 深度代码分析结论存档。**这是学员最大的简历资产，但需要"收复"才能用。**

## 一、基本盘

| 项 | 数据 |
|----|------|
| 源码 | `src/` **28,193 行**（此前误以为 20 万行，那个数字含 dist/build 打包产物） |
| 测试 | `tests/` 22,434 行 / **967 个测试函数**，test:src ≈ 0.8:1 |
| 提交 | 465 次 |
| 形态 | PySide6 桌面版 + CLI + **MCP Server** 三入口共用同一套 contracts→services→tasks 分层 |

**关键处境**：思路和设计是学员提的，代码实现由 AI 完成，学员**未细读**。即：懂 why，不懂 how。而面试深挖恰在 how——**这是当前状态下写进简历会翻车的原因，也是学习的最大动力**。

好消息：28k 行**可以读完**；核心的 `ai_analysis.py`（~800 行）+ `openai_chat.py`（600 行）装着最有价值的部分，一周能吃透。

## 二、真正的 AI 工程能力（可写简历，经得起深挖）

1. **多轮生成-验证-反馈循环**（`ai_analysis.py`）⭐ 最该放简历第一条
   超采样 2× 候选 → LLM 自评分 → **代码用真实字幕 duration 独立复核（不信 LLM 自报）** → 真实数值 + 修正方向回灌下一轮（MAX 3 轮）→ 最终规则排序（**硬约束优先于分数**）；单轮解析失败不作废已积累候选。
   学名：*generate-verify-refine loop with over-sampling + programmatic verifier*，当前 LLM 工程主流范式。
2. **生产级多 provider LLM 网关**（`openai_chat.py`）
   4 种协议（OpenAI/Anthropic/MiniMax/Responses）+ 3 种多模态；指数退避重试含 Cloudflare 5xx；**手写 SSE 解析器**（同时识别 OpenAI delta 与 Anthropic content_block_delta 并合成回统一结构）；**为绕开 CF 100s 超时主动改用流式**；ReadTimeout 明确不重试；**空响应"病历式"诊断**（finish_reason 截断 / 只回 reasoning_content / thinking 块吃光额度三类归因）；thinking 参数 400 兜底；中转站强制 tool_use 的三重降级。
3. **非结构化输出健壮解析层**（`extract_json_response` + 4 个辅助 + 6 个专项测试）
   括号配对状态机 → 正则修尾逗号 → **逐字符修复未转义引号**（向前 probe 判定是否真闭合）→ "别把内层小对象误当结果"的业务键判定。
4. **多模态成本架构**
   `should_call_vision` **只在决策边界带调用**（远高/远低阈值都不花钱）；按 subtitle_id 跨方案两级缓存；**按 80k token 上限反推** 3 分钟切段 + fps 1.0 + 480p 代理转码；磁盘缓存带签名版本。
5. **基于 ASR 字级时间戳的 LLM 决策落地**（`extreme_words.py`）
   检测宽进 → LLM 只返回 `{action: ignore|cut|drop}` → **代码按 word-level timestamp 把违规字从时间轴切除**，切点成为字幕边界，下游零改动。分批防截断 + 缺失决策留痕 + fail-open 风险主动标注。
   **"让 LLM 判断、让代码执行"的职责切分**。
6. **MCP Server 从零手写**（`mcp_server/`，8 个 tool）
   手写 JSON-RPC over stdio；**双 framing 自适应**（换行分隔 JSON + LSP Content-Length）；三层不崩溃保证；返回 content + structuredContent + isError。
7. **LLM 应用可测性架构 + 自建 tracing**
   全链路依赖注入（chat_client / vision_fn / chat_fn / ark_client / detector 全可 fake），967 测试在无 API key、无 cv2、无 mediapipe 环境下可跑；每任务落盘 9 类 LLM 产物（llm_calls / preclean / vision_calls / candidates / selected / raw…），失败路径也落盘。**这就是自建版 Langfuse**。

## 三、只是调服务（含金量有限，别吹）

- **ASR**：调火山引擎（flash 内联 / file+TOS 轮询）。除 ffmpeg 预处理与容量护栏外无技术含量
- **视觉理解**：调豆包方舟 / GPT-4o 类，prompt 与解析是自己的，理解能力是别人的
- **人物检测**：MediaPipe 预训练 `pose_landmarker_lite` 直接 detect，适配层写得细但那是兼容性工程
- **CV 打分**：Laplacian 方差 / 灰度均值 / 阈值像素比 / 帧差 / HSV 直方图——全是 OpenCV 一行调用，加权与阈值是手调经验值
  → 简历措辞：**"基于 OpenCV 的画面质量启发式打分与检索管线"**，⚠️ 不要写"计算机视觉算法"
- `onnxruntime` GPU provider 选择是**死代码**，不能写
- 剪映草稿生成 / ffmpeg 编排 / 授权体系 / PyInstaller 打包：扎实的全栈工程，但与 AI 无关

## 四、必须补的短板（按优先级）

1. 🔴 **零评测体系**——选片质量全靠"实测感觉"，LLM 自评 90 分阈值无 ground truth，程序验证器只覆盖"时长"这一个维度。**面试问"你怎么知道选出来的更好"当前答不上来**
   → **投简历前补最小 eval**：30 条人工标注录像 + "选中片段是否被采纳"命中率，跑基线对比。简历第一条即可升级为「将方案采纳率从 X% 提升到 Y%」
2. 🟡 无 token/成本埋点（成本控制是结构性的，但无法量化）
3. 🟡 Prompt 无版本管理、无 A/B、硬编码在 .py 里
4. 🟡 MCP：`protocolVersion` 硬编码 2024-11-05、无版本协商、无进度通知/取消，`run_draft_workflow` 同步阻塞可跑几十分钟
5. 🟢 CLI 与 MCP 的 30 行装配代码复制粘贴，该提 `build_draft_runtime(settings)`
6. 🟢 大文件：settings.py 1681 / draft_export.py 1545 / export_handlers.py 1395

## 五、学习计划联动（每学一课立刻回读对应实现）

| 阶段 1 课程 | 回读目标 | 产出 |
|---|---|---|
| Chat Completions API | `openai_chat.py` 多 provider 网关 | 能讲清 4 种协议差异 |
| 流式输出与 SSE | `_consume_sse()` 手写解析器 | 能讲"为什么长请求必须走流式"（CF 100s） |
| 结构化输出 | `extract_json_response` 四级修复 | 想清楚"当初为何没用 json_schema"（答辩：中转站支持不一致；但要自己讲得出） |
| 采样参数 | 各 prompt 的温度设定 | — |
| 成本控制 | `should_call_vision` 门控 + 两级缓存 | 能讲 ROI |
| Function Calling | `mcp_server/` 8 个 tool | — |
| 阶段 2 RAG | 素材库/历史爆款检索增强 | **真实产品增值** |
| 阶段 3 MCP | 升级到 2026-07-28 无状态规范 + 进度通知 | 直通选题池 [[开源选题池]] C1 |
| 阶段 4 CV | `cv_frame_analysis.py` 打分与阈值 | 能讲每个阈值的来历 |
| **贯穿** | **补 eval 体系** | 🔴 最高优先级 |

## 六、简历定位

- **不是**"AI 算法项目"，**是**"做得扎实的 LLM 应用工程项目"——价值在"如何让不可靠的 LLM 输出在生产环境稳定产出可用结果"
- 对 LLM Application / AI Infra 方向，**分量足够**；A 组 7 条挑 3 条能撑 40 分钟深挖
- ⚠️ **前提是真的读懂**。当前状态写进简历 = 地雷
- 商业属性（收费、真实用户、465 次提交迭代）本身是强信号，比 star 数硬

## 相关笔记

- [[开源选题池]] · [[学习进度]] · [[02-LLM应用开发-MOC]]
