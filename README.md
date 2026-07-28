# ai-learning-notes — Java 工程师转型 AI 的知识图谱笔记

一套面向 **Java 背景转型者** 的 AI 工程学习笔记，全部由 AI 教练（Claude）按学习进度定制生成、随实际掌握情况迭代。配套实战代码在姊妹仓库 [AiStudy](https://github.com/ChaselT/AiStudy)。

## 特点

- **Java 类比驱动**：每个概念优先用 Java 对照讲解（uv↔Maven、pydantic↔POJO+Bean Validation、装饰器↔注解+AOP、asyncio↔CompletableFuture、Protocol↔interface）
- **Obsidian 知识图谱**：每篇一个概念，`[[双链]]` 互联，图谱视图可视化知识网络生长
- **学-练-考闭环**：每篇笔记带动手任务；每阶段配测试卷（笔试+实操，80 分过关）；错题本记录真实踩坑
- **零基础友好**：Python 部分从"变量不用声明类型"讲起，宁细勿跳

## 目录结构

```
00-总览/        学习地图（总 MOC）、进度追踪、错题与复盘
01-Python基础/  基础篇 9 课（零基础）+ 进阶篇 7 课（差异纠偏与 AI 工程必备）+ 阶段测试
02-LLM应用开发/ Chat API、流式、Prompt 工程、Function Calling、多模态、Ollama、FastAPI
03~08/          RAG、Agent、深度学习、CV、微调、面试——随学习进度生成
```

## 使用方式

用 [Obsidian](https://obsidian.md) 打开本目录（或作为 vault 的子目录），从 `00-总览/AI学习地图.md` 进入；直接当普通 Markdown 阅读也完全可行（GitHub 上双链显示为 `[[名称]]` 纯文本）。

> 技术栈版本基线（2026-07）：LangGraph 1.x、MCP 2026-07-28 规范、Spring AI 2.0（Boot 4 + Java 21）、pydantic v2、uv。
