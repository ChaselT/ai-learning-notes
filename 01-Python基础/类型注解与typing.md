---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# 类型注解与typing

## 这是什么/为什么重要

Python 3.5+ 支持类型注解（type hints）：语法上像 Java 的类型声明 + 泛型，但**运行时默认不强制**，靠 mypy/pyright 等工具做静态检查。现代 AI 工程代码（FastAPI、LangChain、pydantic、openai SDK）几乎 100% 带类型注解——FastAPI 甚至靠注解生成接口文档和参数校验。对 Java 工程师来说这是最友好的特性：把静态类型的安全感找回来。

## 核心内容

### 基本注解

```python
def greet(name: str, times: int = 1) -> str:
    return ("Hello, " + name + "! ") * times

age: int = 30            # 变量注解（局部变量通常省略，靠推断）
# Java: String greet(String name, int times)
# 区别：注解只是元数据，greet(123, "x") 运行时不报错，靠工具检查
```

### 容器泛型：list[str] / dict[str, int]

Python 3.9+ 直接用内置类型做泛型（不再需要 `typing.List`）：

```python
def word_count(words: list[str]) -> dict[str, int]:
    counts: dict[str, int] = {}
    for w in words:
        counts[w] = counts.get(w, 0) + 1
    return counts

pairs: list[tuple[str, float]] = [("loss", 0.32), ("acc", 0.91)]
# Java: List<String>, Map<String, Integer> —— 一一对应，语法更短
```

### Optional 与 Union

```python
def find_user(uid: int) -> str | None:      # 3.10+ 写法，等价 Optional[str]
    return "alice" if uid == 1 else None

def parse(v: int | str) -> int:             # Union：接受多种类型
    return v if isinstance(v, int) else int(v)

user = find_user(2)
if user is not None:        # 类型检查器会做 narrowing，之后 user 就是 str
    print(user.upper())
# Java 对比：str | None ≈ Optional<String>，但 Python 靠检查器逼你判空，
# 而不是包一层容器对象
```

### TypedDict：给 dict 定结构

处理 JSON（LLM API 响应）时大量用 dict，`TypedDict` 给它加上「schema」：

```python
from typing import TypedDict

class ChatMessage(TypedDict):
    role: str        # "system" | "user" | "assistant"
    content: str

msg: ChatMessage = {"role": "user", "content": "hi"}
# msg = {"role": "user"}  ← 检查器报错：缺少 content
# 类比：给 Map<String, Object> 换成一个轻量 DTO，但运行时仍是普通 dict
```

需要运行时真校验时升级到 pydantic（见[[dataclass与pydantic]]）。

### Protocol：结构化接口（类比 interface）

Java 的 interface 要显式 `implements`；Python 的 `Protocol` 只看结构，是[[Python与Java核心差异]]里鸭子类型的静态化：

```python
from typing import Protocol

class Embedder(Protocol):
    def embed(self, text: str) -> list[float]: ...

class OpenAIEmbedder:            # 没有继承 Embedder！
    def embed(self, text: str) -> list[float]:
        return [0.1, 0.2]

def index_doc(embedder: Embedder, text: str) -> list[float]:
    return embedder.embed(text)

index_doc(OpenAIEmbedder(), "hello")   # 结构匹配即通过类型检查
```

这是 AI 工程解耦的常用手段：定义 `Embedder`/`VectorStore` 等 Protocol，实现类随便换。

### mypy / pyright 静态检查

```powershell
uv add --dev mypy
uv run mypy .                # 检查整个项目
```

```python
def double(x: int) -> int:
    return x * 2

double("oops")   # mypy: Argument 1 to "double" has incompatible type "str"
```

- **mypy**：老牌命令行检查器，CI 里常用
- **pyright**：微软出品，VS Code 的 Pylance 插件内置的就是它——在设置里把 `python.analysis.typeCheckingMode` 调成 `basic` 或 `strict`，即可获得接近 Java 编译器的实时报错体验

工程习惯：**所有函数签名必须写注解**，局部变量靠推断；这也是给 AI 编程助手最有效的上下文。

## 动手任务

1. 写 `E:\workspace\AiStudy\phase0-python\typing_basics.py`：实现 `parse_scores(raw: list[str]) -> dict[str, float]`（输入形如 `["math:90.5", "en:80"]`），全程带注解，用 `uv run mypy typing_basics.py` 检查到零报错。
2. 写 `E:\workspace\AiStudy\phase0-python\typing_protocol.py`：定义 `Notifier(Protocol)`（含 `send(msg: str) -> bool`），实现 `ConsoleNotifier` 和 `FileNotifier` 两个类（都不继承 Protocol），写 `broadcast(notifiers: list[Notifier], msg: str) -> None` 依次调用。
3. 故意在上题中把 `send` 的返回类型改成 `str`，观察 mypy/Pylance 的报错信息，记录到注释里。

## 相关笔记

- [[01-Python基础-MOC]]
- [[uv与项目管理]] —— 上一篇：mypy 用 uv 安装
- [[dataclass与pydantic]] —— 下一篇：注解从「检查」升级到「运行时校验」
- [[Python与Java核心差异]] —— 鸭子类型 → Protocol 的来龙去脉
