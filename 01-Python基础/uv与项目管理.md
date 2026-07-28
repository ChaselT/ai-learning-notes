---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# uv与项目管理

## 这是什么/为什么重要

uv 是 Rust 写的新一代 Python 包与项目管理器，**定位相当于 Maven/Gradle + SDKMAN 的合体**：管依赖、管虚拟环境、甚至管 Python 版本本身，速度比传统 pip 快 10-100 倍。2024 年后新项目基本都用 uv，AI 生态（LangChain、FastAPI 项目模板）也在全面转向它。先把它装好，本阶段所有练习代码都用 uv 管理。

## 核心内容

### 安装 uv（Windows PowerShell）

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# 装完重开终端，验证：
uv --version
```

uv 还能替你装 Python 本体（类似 SDKMAN 管 JDK）：

```powershell
uv python install 3.12
uv python list        # 查看已安装的 Python 版本
```

### 创建项目：uv init

> [!important] 本项目已经建好，这一节只需要看懂，不要执行
> `E:\workspace\AiStudy\phase0-python\` 已由 Claude 在 2026-07-24 初始化完成（Python **3.11**，已装 httpx），**不要再执行 `uv init`**。下面的命令是给你理解流程用的，将来开新阶段项目（如 phase1-llm-api）时才会真正用到。

```powershell
# 通用流程示例（本阶段无需执行）
cd E:\workspace\AiStudy
uv init my-new-project --python 3.12    # 版本按需选，本项目用的是 3.11
cd my-new-project
```

生成的结构：

```text
phase0-python/
├── pyproject.toml   # ≈ pom.xml：项目元数据 + 依赖声明
├── .python-version  # 锁定 Python 版本（≈ .sdkmanrc / toolchain）
├── main.py
└── README.md
```

### pyproject.toml 类比 pom.xml

```toml
[project]
name = "phase0-python"          # ≈ <artifactId>
version = "0.1.0"               # ≈ <version>
requires-python = ">=3.12"      # ≈ maven.compiler.release
dependencies = [                # ≈ <dependencies>
    "httpx>=0.27",
    "pydantic>=2.7",
]

[dependency-groups]             # ≈ <scope>test</scope> 之类的分组
dev = ["pytest>=8.0", "mypy>=1.10"]
```

对照记忆：`pyproject.toml` = `pom.xml`，`uv.lock` = 依赖锁定文件（Maven 没有直接对应物，类似 npm 的 `package-lock.json`，保证团队装到完全相同的版本）。

### 加依赖：uv add

```powershell
uv add httpx pydantic          # 加运行依赖，自动写入 pyproject.toml 并更新 uv.lock
uv add --dev pytest mypy       # 加开发依赖（≈ test/provided scope）
uv remove httpx                # 移除依赖
uv sync                        # 按 lock 文件还原环境（≈ mvn install 拉依赖，clone 项目后第一步）
```

### 运行：uv run

```powershell
uv run main.py                 # 在项目虚拟环境中运行脚本
uv run pytest                  # 运行环境里装的命令行工具
uv run python -c "import httpx; print(httpx.__version__)"
```

`uv run` 会自动确保虚拟环境存在且与 lock 文件一致，**不需要手动激活环境**——这是和老式工作流最大的体验差异。

### 虚拟环境是什么

Java 里依赖装在项目的 classpath / 本地仓库，天然按项目隔离；Python 的 `pip install` 默认装到**全局解释器**，多项目共用会版本冲突。虚拟环境（venv）就是给每个项目一份独立的解释器 + site-packages 目录，实现 Java 那种「按项目隔离依赖」的效果。

uv 把它自动化了：`uv add` / `uv run` 会自动创建和使用项目根目录下的 `.venv/`，你几乎感觉不到它的存在。VS Code 里选择解释器时指向 `.venv\Scripts\python.exe` 即可。

### 为什么不再直接用 pip

| 直接用 pip 的问题 | uv 的解法 |
| --- | --- |
| 默认装到全局，项目间互相污染 | 自动创建/使用项目级 `.venv` |
| `requirements.txt` 手工维护，无锁定 | `pyproject.toml` 声明 + `uv.lock` 精确锁定 |
| 不管 Python 版本本身 | `uv python install` 统一管理 |
| 解析慢 | Rust 实现，秒级完成 |

结论：`pip install xxx` 在教程里还会见到，动手时一律换成 `uv add xxx`。

## 动手任务

1. 项目已建好（见上方提示），你的任务是**验证并看懂它**：进入 `E:\workspace\AiStudy\phase0-python\`，运行 `uv run main.py` 确认输出；打开 `pyproject.toml` 和 `.python-version`，对照本笔记说出每一段是干什么的。
2. 执行 `uv add pydantic` 与 `uv add --dev mypy pytest`（httpx 已装过），观察 `pyproject.toml` 的变化，然后写一份 `E:\workspace\AiStudy\phase0-python\notes_pyproject.md`，逐段注释它与 `pom.xml` 的对应关系。
3. 写 `E:\workspace\AiStudy\phase0-python\check_env.py`：打印 `sys.executable` 和 `httpx.__version__`，用 `uv run check_env.py` 运行，确认解释器路径指向项目的 `.venv`。

## 相关笔记

- [[01-Python基础-MOC]]
- [[Python与Java核心差异]] —— 上一篇
- [[类型注解与typing]] —— 下一篇：mypy 就是这里 `uv add --dev` 装的
- [[Jupyter与调试]] —— notebook 环境同样由 uv 管理
