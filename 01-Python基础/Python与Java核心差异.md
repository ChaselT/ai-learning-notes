---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# Python与Java核心差异

## 这是什么/为什么重要

你已经会编程，学 Python 最大的风险不是「学不会」，而是**用 Java 的心智模型硬套 Python**，写出能跑但很别扭、甚至暗藏 bug 的代码。这篇笔记集中列出两门语言在类型系统、执行模型、工程约定上的核心差异和高频踩坑点，先建立正确的直觉，后面每篇笔记都会更顺。

## 核心内容

### 动态类型：类型跟着值走，不跟着变量走

```python
x = 1          # x 此刻指向 int
x = "hello"    # 合法！变量只是名字（引用），没有声明类型
# Java: int x = 1; x = "hello";  → 编译错误
```

Python 是**强类型 + 动态类型**：`"1" + 1` 会抛 `TypeError`（强类型，不像 JS 自动转换），但类型检查发生在运行时。工程上靠[[类型注解与typing]]找回静态检查。

### 缩进即语法，没有大括号

```python
def grade(score: int) -> str:
    if score >= 60:
        return "pass"      # 缩进决定代码块，约定 4 空格
    return "fail"          # 没有 {}，没有分号
```

坑：混用 Tab 和空格会报 `IndentationError`。编辑器统一设成 4 空格即可。

### 一切皆对象

函数、类、模块本身都是对象，可以赋值、传参、放进 dict：

```python
def hello() -> None:
    print("hi")

f = hello          # 函数赋值给变量（Java 要用方法引用/函数式接口才能做到）
f()                # hi
print(type(3))     # <class 'int'>，连 int 都是对象，没有基本类型/装箱之分
```

这是[[装饰器与上下文管理器]]的基础：装饰器就是「接收函数、返回函数」的函数。

### 鸭子类型：看行为不看类型

```python
class Duck:
    def speak(self) -> str: return "quack"

class Dog:
    def speak(self) -> str: return "woof"

def make_sound(animal) -> None:
    print(animal.speak())   # 不要求实现某个接口，有 speak 方法就行

make_sound(Duck()); make_sound(Dog())
# Java 思维：必须先定义 interface Speakable。Python 里接口是隐式的
```

需要显式契约时用 `Protocol`（见[[类型注解与typing]]）。

### GIL：多线程 ≠ 多核并行

CPython 有全局解释器锁（GIL）：**同一时刻只有一个线程执行 Python 字节码**。

- CPU 密集任务：`threading` 无法利用多核，要用 `multiprocessing`（多进程）
- IO 密集任务（调 LLM API、读写网络）：GIL 会在 IO 等待时释放，多线程有效，但更主流的方案是 `asyncio`（见[[async异步编程]]）
- Java 对比：Java 线程是真并行；Python 里别把 `threading` 当 Java 线程池用

### 包与模块 vs Java package

- 一个 `.py` 文件就是一个 **module**；含 `__init__.py` 的目录是一个 **package**
- `import` 的是文件/目录路径，不像 Java 强制「目录结构 = 域名倒写」
- 没有 `public/private` 关键字：约定 `_name` 表示内部使用，`__name__` 触发名称改写（近似 private）

```python
from pathlib import Path      # 从 pathlib 模块导入 Path 类
import json                   # 导入整个模块，用 json.loads 调用
```

### 命名规范（PEP 8）

| Java | Python |
| --- | --- |
| `getUserName()` | `get_user_name()`（snake_case） |
| `UserService` | `UserService`（类名同样 PascalCase） |
| `MAX_SIZE` | `MAX_SIZE`（常量相同） |
| `private int count` | `self._count`（约定，无强制） |

### 高频踩坑

**坑 1：可变默认参数只求值一次**

```python
def add_item(item: str, items: list[str] = []) -> list[str]:  # 错误写法！
    items.append(item)
    return items

add_item("a")   # ['a']
add_item("b")   # ['a', 'b'] —— 默认列表在函数定义时创建一次，被所有调用共享

def add_item_ok(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []          # 正确姿势：默认值用 None，函数体内新建
    items.append(item)
    return items
```

**坑 2：`is` vs `==`**

```python
a = [1, 2]; b = [1, 2]
print(a == b)   # True，值相等（相当于 Java 的 equals）
print(a is b)   # False，不是同一个对象（相当于 Java 的 ==）
# 规则：只有和 None 比较时才用 is：if x is None
```

注意 Java 直觉在这里是**反的**：Java 的 `==` 比引用，Python 的 `==` 比值。

## 动手任务

1. 在 `E:\workspace\AiStudy\phase0-python\diff_basics.py` 中：写一个函数演示可变默认参数的坑（连续调用两次打印结果），再写修复版；并打印 `a == b` 与 `a is b` 对小整数（如 5）和列表的不同表现。
2. 在 `E:\workspace\AiStudy\phase0-python\duck_typing.py` 中：定义两个没有共同父类的类 `JsonReport` 和 `TextReport`，都实现 `render() -> str`，写一个 `print_report(report)` 函数依次处理它们，体会鸭子类型。
3. 口头/笔记作答：为什么 Python 里调 10 个 LLM API 用 `asyncio` 而不是开 10 个线程？（写 3 句话，学完[[async异步编程]]后回来修订）

## 相关笔记

- [[01-Python基础-MOC]]
- [[类型注解与typing]] —— 下一篇（进阶篇 2/7）：动态类型的工程化补救
- [[uv与项目管理]] —— 已完成（基础篇第 0 课）
- [[类型注解与typing]] —— 动态类型的工程化补救
- [[async异步编程]] —— GIL 话题的延续
- [[常用标准库与生态]] —— 写出 Pythonic 代码
