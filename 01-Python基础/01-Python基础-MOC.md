---
tags: [AI学习, 阶段0, Python, MOC]
created: 2026-07-24
---

# 01-Python基础 MOC

> 阶段 0：作为 Java 工程师，用最短路径把 Python 补到「能写 AI 工程代码」的水平。
> 上级地图：[[AI学习地图]]

## 这是什么/为什么重要

这是 Python 基础阶段的总地图（Map of Content）。你已经有多年 Java 经验，不需要从「什么是变量」学起，这个阶段只解决一件事：**把 Java 心智模型映射/纠偏到 Python**，并掌握 AI 工程中高频出现的语言特性（类型注解、pydantic、async、装饰器）。后续 LangChain、FastAPI、RAG 阶段全部建立在这些内容之上。

## 学习顺序

零基础起步，分两轮。每篇学完立刻做「动手任务」：基础篇代码写到 `E:\workspace\AiStudy\phase0-python\basics\`，进阶篇写到 `E:\workspace\AiStudy\phase0-python\`。

### 第一轮：基础篇（第 1-1.5 周，从零学语法）

| 顺序  | 笔记               | 一句话简介                                    |
| --- | ---------------- | ---------------------------------------- |
| 0   | [[uv与项目管理]]      | 先把环境和运行方式搞定（uv 类比 Maven），能跑 `uv run` 再开始 |
| 1   | [[基础01-变量与基本类型]] | 变量不用声明类型、print/f-string、None vs null     |
| 2   | [[基础02-字符串]]     | 切片语法（Java 没有的重点）、常用方法、不可变性               |
| 3   | [[基础03-列表与元组]]   | 类比 ArrayList/数组：增删改查、切片、元组解包             |
| 4   | [[基础04-字典与集合]]   | 类比 HashMap/HashSet：取值行为差异是重点             |
| 5   | [[基础05-控制流]]     | for-in 不是 C 风格 for、真值测试（空即假）、match-case  |
| 6   | [[基础06-函数]]      | 默认参数、关键字参数、*args/**kwargs、没有重载怎么办        |
| 7   | [[基础07-类与对象]]    | self 讲透（最别扭的点）、__init__、继承、@property     |
| 8   | [[基础08-异常处理]]    | try/except、EAFP 文化差异                     |
| 9   | [[基础09-模块与导入]]   | import、`if __name__ == "__main__"` 讲透    |

### 第二轮：进阶篇（第 1.5-3 周，差异纠偏 + AI 工程必备）

| 顺序 | 笔记 | 一句话简介 |
| --- | --- | --- |
| 10 | [[Python与Java核心差异]] | 学完基础再看：GIL、鸭子类型、高频踩坑总复习 |
| 11 | [[类型注解与typing]] | 类比泛型与接口：AI 工程代码几乎全带类型注解 |
| 12 | [[dataclass与pydantic]] | 类比 POJO/Lombok：pydantic 是 LLM 结构化输出的基石 |
| 13 | [[装饰器与上下文管理器]] | 类比注解+AOP、try-with-resources |
| 14 | [[async异步编程]] | 类比 CompletableFuture：并发调 LLM API 的前提 |
| 15 | [[常用标准库与生态]] | pathlib/httpx/logging/推导式：Pythonic 工具箱 |
| 16 | [[Jupyter与调试]] | AI 实验标配：notebook 工作流与断点调试 |

## 阶段验收标准

全部满足才进入阶段 1：

- [ ] 能用 `uv init` 创建项目、`uv add` 加依赖、`uv run` 运行脚本，说得清 `pyproject.toml` 里每一段是干什么的
- [ ] 能独立写一个**带完整类型注解的异步脚本**：用 `httpx.AsyncClient` + `asyncio.gather` 并发请求多个 URL，结果用 `pydantic.BaseModel` 解析校验，`mypy`/`pyright` 检查零报错
- [ ] 能口头解释：GIL 对多线程的影响、可变默认参数的坑、`is` 与 `==` 的区别、为什么 LLM 应用离不开 async
- [ ] 能在 VS Code 中新建 `.ipynb` 做实验，并用断点调试一个普通 `.py` 脚本
- [ ] **通过 [[阶段0测试]]（笔试+实操，Claude 批改 ≥80 分）——这是进入阶段 1 的硬门槛**

## 本阶段全部笔记

- [[Python与Java核心差异]]
- [[uv与项目管理]]
- [[类型注解与typing]]
- [[dataclass与pydantic]]
- [[装饰器与上下文管理器]]
- [[async异步编程]]
- [[常用标准库与生态]]
- [[Jupyter与调试]]

## 通向下一阶段

学完后进入阶段 1（LLM 应用基础），本阶段内容会直接支撑：

- [[结构化输出]]（依赖 pydantic）
- [[流式输出与SSE]]（依赖 async）
