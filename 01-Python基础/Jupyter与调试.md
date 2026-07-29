---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# Jupyter与调试

## 这是什么/为什么重要

Java 开发的节奏是「写完 → 编译 → 跑整个程序」；AI 开发大量时间花在**试**：试一个 prompt、看一个 DataFrame、调一个 API 参数。Jupyter Notebook 提供「代码分块执行、变量常驻内存、结果就地显示」的交互式工作流，是 AI 实验的标配——几乎所有模型厂商的官方示例都是 `.ipynb`。这一篇同时补上 Python 断点调试，让你把 IDEA 的调试习惯迁移到 VS Code。

## 核心内容

### cell 执行模型

Notebook 由一个个 cell 组成，背后是一个常驻的 Python 进程（kernel）：

- 每个 cell 可独立、反复执行；**所有 cell 共享同一份全局变量**
- cell 最后一个表达式的值自动显示，不用 print
- 执行顺序由你点击的顺序决定，与 cell 排列顺序无关——这是双刃剑

```python
# cell 1
import httpx
data = httpx.get("https://httpbingo.org/json").json()   # 只需请求一次

# cell 2（反复修改、反复运行，data 一直在内存里）
data["slideshow"]["title"]     # 值直接显示在 cell 下方
```

对 Java 工程师的价值：加载模型/拉数据这类要几十秒的操作只做一次，之后无限次快速迭代后续逻辑——相当于一个可保存的、增强版 JShell。

**乱序执行的坑**：改了上游 cell 却没重跑下游，状态就和代码对不上。出现诡异结果时，第一反应是 **Restart Kernel + Run All**，验证笔记本能从头到尾干净地跑通。

### VS Code 中使用 .ipynb

1. 安装扩展：**Python** + **Jupyter**（微软官方）
2. 项目里加依赖：`uv add --dev ipykernel`
3. 新建 `experiment.ipynb`，右上角 **Select Kernel** → 选择项目的 `.venv\Scripts\python.exe`（这样 notebook 能 import 你 `uv add` 的所有包）
4. 常用快捷键：`Shift+Enter` 运行当前 cell 并跳到下一个；`Esc` 后 `A`/`B` 在上/下方插入 cell，`DD` 删除 cell

轻量替代：在普通 `.py` 文件里写 `# %%` 分隔符，VS Code 会把它渲染成可单独运行的 cell（Interactive Window）——兼得脚本的 git 友好和 notebook 的交互性。

```python
# %%
nums = [1, 2, 3]

# %%
sum(nums)      # 单独运行这一块
```

### 什么时候用 notebook，什么时候用脚本

| 场景 | 选择 |
| --- | --- |
| 试 prompt、探索数据、调参、学新库 API | notebook |
| 结果要配图表/说明分享给别人 | notebook |
| 会被反复运行、被 import、进 CI、上生产 | 脚本 / 模块 |
| 逻辑稳定后 | 从 notebook **抽函数**进 `.py`，notebook 只留调用和展示 |

纪律：notebook 是实验室，不是交付物。业务逻辑一旦成型就搬进带类型注解的模块（见[[类型注解与typing]]），notebook 里 `from mylib import xxx` 来用。`.ipynb` 是 JSON 文件，diff 很难读，尽量别在里面沉淀核心代码。

### 断点调试：把 IDEA 的习惯搬过来

**方式 1：VS Code 图形化调试（最接近 IDEA）**

1. 打开 `.py` 文件，行号左侧点击设断点
2. `F5` → 选择 "Python File"（uv 项目会自动使用 `.venv` 解释器）
3. `F10` step over、`F11` step into、`F5` continue；左侧看变量，Debug Console 里可以执行任意表达式（≈ IDEA 的 Evaluate Expression）

复杂项目可在 `.vscode/launch.json` 配置入口、参数和环境变量。notebook 里同样支持：cell 左侧下拉选 **Debug Cell** 即可带断点执行。

**方式 2：代码内断点 breakpoint()**

```python
def process(items: list[int]) -> int:
    total = 0
    for i, item in enumerate(items):
        breakpoint()          # 运行到此进入 pdb 交互调试器
        total += item
    return total
```

pdb 常用命令：`n`（下一行）、`s`（步入）、`c`（继续）、`p total`（打印变量）、`ll`（当前函数源码）、`q`（退出）。适合没有 IDE 的场景（远程服务器、容器里）。

**方式 3：日志调试**：并发/async 代码里断点会打乱时序，配置好 logging（见[[常用标准库与生态]]）+ `f"{var=}"` 打印往往更高效。

## 动手任务

1. 在 `E:\workspace\AiStudy\phase0-python\` 下执行 `uv add --dev ipykernel`，创建 `experiment.ipynb`：cell 1 用 httpx 请求 `https://httpbingo.org/json` 存入变量；cell 2、3 分别对该变量做不同处理，体会「请求一次、反复实验」；最后 Restart + Run All 验证可复现。
2. 写 `E:\workspace\AiStudy\phase0-python\debug_practice.py`：实现一个故意有 bug 的函数（如对 `[3, 1, 4, 1, 5]` 求最大值但初始值写成 `0` 且列表可能全为负数），用 VS Code 断点单步定位 bug，修复后在注释里记录定位过程。
3. 把任务 1 notebook 中的处理逻辑抽成函数，移入 `E:\workspace\AiStudy\phase0-python\explore_utils.py`（带类型注解），notebook 改为 import 调用——演练一遍「notebook 实验 → 沉淀为模块」的完整流程。

## 相关笔记

- [[01-Python基础-MOC]]
- [[常用标准库与生态]] —— 上一篇：logging 与 httpx 在这里都用到了
- [[uv与项目管理]] —— ipykernel 也由 uv 管理
- [[类型注解与typing]] —— 沉淀为模块时补全注解
- [[AI学习地图]] —— 阶段 0 到此收尾，回地图看阶段 1
