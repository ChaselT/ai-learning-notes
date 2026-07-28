---
tags: [AI学习, 阶段0, Python]
created: 2026-07-24
---

# dataclass与pydantic

## 这是什么/为什么重要

Java 里你写 POJO 靠 Lombok 消灭样板代码，靠 Bean Validation（`@NotNull`、`@Size`）做校验，靠 Jackson 做 JSON 序列化——Python 里对应物是：`dataclass`（≈ Lombok 的 `@Data`）和 **pydantic**（≈ Lombok + Bean Validation + Jackson 三合一）。pydantic 是当前 AI 工程的绝对核心：FastAPI 的请求/响应模型、LangChain 的配置对象、LLM 结构化输出全都基于它，必须练熟。

## 核心内容

### dataclass：标准库版 @Data

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int = 0                                  # 默认值
    tags: list[str] = field(default_factory=list) # 可变默认值必须用 factory！

u = User(name="alice", age=30)
print(u)              # User(name='alice', age=30, tags=[]) —— 自动 __repr__
print(u == User("alice", 30))   # True —— 自动 __eq__（≈ Lombok @EqualsAndHashCode）
```

自动生成 `__init__`/`__repr__`/`__eq__`，相当于 Lombok 的 `@Data` + `@AllArgsConstructor`。注意 `field(default_factory=list)` 正是[[Python与Java核心差异]]里「可变默认参数」坑的官方解法。

**局限**：dataclass 不做任何校验和类型转换，`User(name=123, age="x")` 照样创建成功。它适合纯内部数据结构；一旦数据来自外部（JSON、API、LLM），就该用 pydantic。

### pydantic BaseModel：运行时真校验

```python
from pydantic import BaseModel, Field, ValidationError

class Article(BaseModel):
    title: str = Field(min_length=1, max_length=100)   # ≈ @Size(min=1, max=100)
    views: int = Field(ge=0)                           # ≈ @Min(0)
    tags: list[str] = []                               # pydantic 会深拷贝，无共享坑

a = Article(title="Hello", views="42")   # "42" 被自动转换为 int 42（类型转换）
print(a.views + 1)                       # 43

try:
    Article(title="", views=-1)
except ValidationError as e:
    print(e)   # 两条错误一起报出：title 太短、views < 0
```

对比 Bean Validation：注解写在 `Field(...)` 参数里；校验在**构造时自动触发**，不需要 Validator 手动调用；错误信息结构化、可直接返回给前端（FastAPI 就是这么做的）。

### 自定义校验：field_validator

```python
from pydantic import BaseModel, field_validator

class Signup(BaseModel):
    email: str

    @field_validator("email")
    @classmethod
    def must_contain_at(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("invalid email")
        return v.lower()          # 还能顺手做规范化
```

### 序列化与反序列化：model_dump / model_validate

对应 Jackson 的 `writeValueAsString` / `readValue`：

```python
a = Article(title="Hello", views=42)

data = a.model_dump()            # → dict：{'title': 'Hello', 'views': 42, 'tags': []}
json_str = a.model_dump_json()   # → JSON 字符串

b = Article.model_validate({"title": "Hi", "views": 1})   # dict → 对象（带校验）
c = Article.model_validate_json('{"title": "Hi", "views": 1}')  # JSON 字符串 → 对象
```

模型可以嵌套，pydantic 会递归校验：

```python
class Author(BaseModel):
    name: str

class Post(BaseModel):
    author: Author
    title: str

p = Post.model_validate({"author": {"name": "bob"}, "title": "t"})
print(p.author.name)   # bob —— 嵌套 dict 自动转成 Author 对象
```

### 为什么 LLM 结构化输出全靠 pydantic

LLM 返回的是文本/JSON，天生不可靠：字段可能缺、类型可能错。工程上的标准做法：

1. 用 `BaseModel` 定义期望的输出 schema
2. 把 schema（`Article.model_json_schema()` 生成的 JSON Schema）塞给 LLM API，约束它按格式输出
3. 拿到响应后 `model_validate_json` 一步完成解析 + 校验，失败就重试

OpenAI 的 structured outputs、LangChain 的 `with_structured_output()`、Instructor 库，底层全是这套流程。所以 pydantic 熟练度直接决定你写 LLM 应用的效率——详见阶段 1 的 [[结构化输出]]。

### 选型口诀

- 纯内部传递、不出进程边界 → `dataclass`（零依赖、更轻）
- 数据跨越信任边界（HTTP、LLM、配置文件、数据库）→ `pydantic`

## 动手任务

1. 写 `E:\workspace\AiStudy\phase0-python\models_dataclass.py`：用 dataclass 定义 `Point(x: float, y: float)`，实现 `distance(p1, p2) -> float`，验证自动 `__eq__` 行为。
2. 写 `E:\workspace\AiStudy\phase0-python\models_pydantic.py`：定义 `Movie(title, year, rating)`，约束 `1900 <= year <= 2030`、`0 <= rating <= 10`；构造一组合法/非法数据，打印 `ValidationError` 的内容；再演示 `model_dump_json` → `model_validate_json` 的往返。
3. 写 `E:\workspace\AiStudy\phase0-python\llm_schema.py`：定义嵌套模型 `Recipe(name: str, ingredients: list[Ingredient])`，打印 `Recipe.model_json_schema()` 的输出，观察这份 JSON Schema——这就是将来喂给 LLM 的格式约束。

## 相关笔记

- [[01-Python基础-MOC]]
- [[类型注解与typing]] —— 上一篇：pydantic 的字段定义就是类型注解
- [[装饰器与上下文管理器]] —— 下一篇：`@dataclass`、`@field_validator` 本身就是装饰器
- [[结构化输出]] —— 阶段 1：pydantic 在 LLM 工程中的主战场
