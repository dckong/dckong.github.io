---
title: "Pydantic 入门笔记：让外部数据变成可靠的 Python 对象"
date: 2026-09-09 08:00:00 +0800
categories: [Dev, Python]
tags: [python, pydantic, beginner, learning-notes]
description: 从读书计划小案例出发，理解类型提示、数据校验、默认值、验证器和 JSON，附可运行代码与练习答案。
toc: true
---

这篇笔记适合已经会写函数、字典和简单类，但还没用过 Pydantic 的读者。建议用 45 分钟完成：先运行例子，再修改输入，最后做练习。环境约定为 **Python 3.11+、Pydantic 2.x**。

学习起点是 Real Python 的 [Pydantic 教程](https://realpython.com/python-pydantic/)。原文从数据校验讲到模型、字段、自定义验证和函数参数验证。这里按初学者的操作顺序重新组织，使用自行编写的“读书计划”案例；API 细节另对照官方文档。本文是学习笔记，不是原文翻译。

系列下一篇：[asyncio 入门笔记]({% post_url /dev/python/2026-09-09-asyncio-beginner-notes %})。

## 1. 先看问题：字典里有数据，不等于数据能用

假设网页表单给你一条读书计划：

```python
raw = {"title": "Python 小练习", "pages": "120"}
```

你想计算剩余页数，却发现 `raw["pages"] - 10` 会报错，因为 `"120"` 是字符串。你可以在各处手写转换、判空、范围检查；字段多了之后，很容易漏掉某个入口。

**Pydantic 模型就是集中声明输入规则的地方。** 数据进入时先验证，后续代码再使用模型对象。验证通过只说明符合你写下的规则，并不证明书名真实、用户有权限或数据库中没有重复记录。

普通 Python 类型提示也不会自动执行这些检查：

```python
pages: int = "120"
print(type(pages).__name__)  # str；类型提示没有把它变成整数
```

## 2. 第一个模型：运行，再故意让它失败

在你选定的 Python 3.11+ 环境中安装：

```bash
python -m pip install "pydantic>=2,<3"
```

保存下面的完整代码为 `first_model.py`，运行 `python first_model.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class ReadingPlan(BaseModel):
    title: str
    pages: int = Field(gt=0)
    finished: bool = False


plan = ReadingPlan.model_validate({"title": "Python 小练习", "pages": "120"})
print(plan.pages, type(plan.pages).__name__)
print(plan.finished)

try:
    ReadingPlan(title="Python 小练习", pages=0)
except ValidationError as error:
    for item in error.errors():
        print(item["loc"], item["type"])
```

预期输出：

```text
120 int
False
('pages',) greater_than
```

逐步理解这个过程：继承 `BaseModel` 定义模型；`pages: int` 声明目标类型；`Field(gt=0)` 加上正数约束；`model_validate()` 接收字典并返回实例。默认模式允许一些转换，所以 `"120"` 能成为 `120`。`0` 虽然是整数，却不符合正数规则。

读取错误时先看 `loc`（哪个位置），再看 `type`（错误类别），需要解释时看 `msg`。不要依赖整段英文报错完全不变。模型基础和嵌套模型见 [官方 Models 文档](https://docs.pydantic.dev/latest/concepts/models/)。

## 3. 三种“可选”必须分清

下面是独立的声明示例，不需要接在上一段后面运行：

```python
from pydantic import BaseModel


class RequiredNote(BaseModel):
    note: str              # 必须传入，不能为 None


class NullableNote(BaseModel):
    note: str | None       # 必须传入，但可以明确传入 None


class OptionalNote(BaseModel):
    note: str | None = None # 可以省略，省略时采用 None
```

**“允许空值”和“允许省略”是两件事。** 在 Pydantic 2 中，`str | None` 本身不会提供默认值。字段约定见 [官方 Fields 文档](https://docs.pydantic.dev/latest/concepts/fields/)。

## 4. 用一个完整案例把规则串起来

现在把需求写清楚：一本书有总页数和已读页数；标题不能只有空格；已读页数不能超过总页数；每个计划有独立 ID；意外多传的字段要被发现。

保存为 `reading_plan.py`，运行 `python reading_plan.py`。本例只需要 Pydantic，不需要联网或申请 API Key。

```python
from typing import Self
from uuid import UUID, uuid4
from pydantic import (
    BaseModel, ConfigDict, Field, ValidationError,
    field_validator, model_validator,
)


class ReadingPlan(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    plan_id: UUID = Field(default_factory=uuid4)
    title: str = Field(min_length=1)
    pages: int = Field(gt=0)
    completed_pages: int = Field(default=0, ge=0)
    tags: list[str] = Field(default_factory=list)
    note: str | None = None

    @field_validator("title")
    @classmethod
    def reject_placeholder(cls, value: str) -> str:
        if value == "待定":
            raise ValueError("请填写具体书名")
        return value

    @model_validator(mode="after")
    def check_progress(self) -> Self:
        if self.completed_pages > self.pages:
            raise ValueError("已读页数不能超过总页数")
        return self


if __name__ == "__main__":
    plan = ReadingPlan.model_validate({
        "title": "  Python 小练习  ",
        "pages": "120",
        "completed_pages": 30,
    })
    print(plan.title)
    print(f"剩余 {plan.pages - plan.completed_pages} 页")
    print(plan.model_dump(exclude={"plan_id"}))

    another = ReadingPlan(title="异步编程练习", pages=80)
    print("不同 ID:", plan.plan_id != another.plan_id)

    try:
        ReadingPlan(title="Python 小练习", pages=120, completed_pages=121)
    except ValidationError as error:
        print(error.errors()[0]["msg"])
```

预期输出：

```text
Python 小练习
剩余 90 页
{'title': 'Python 小练习', 'pages': 120, 'completed_pages': 30, 'tags': [], 'note': None}
不同 ID: True
Value error, 已读页数不能超过总页数
```

### 字段约束管单项，模型验证器管关系

`gt=0` 表示大于零，`ge=0` 表示大于等于零。`str_strip_whitespace=True` 先去掉字符串两端空白，所以标题 `"   "` 无法通过长度检查。`extra="forbid"` 能帮你发现拼错的字段名，例如 `page`；默认情况下额外字段会被忽略。

`field_validator` 处理一个字段的自定义规则，这里拒绝占位书名。默认的 `after` 模式在字段类型校验后运行；返回值会成为字段值，因此别忘记 `return value`。`model_validator(mode="after")` 读取完成字段校验的实例，适合比较两项数据，并需要返回 `self`。详见 [官方 Validators 文档](https://docs.pydantic.dev/latest/concepts/validators/)。

### 默认工厂为什么不加括号

`default_factory=uuid4` 把函数交给模型，每次需要默认值时再调用。若写成 `plan_id: UUID = uuid4()`，调用会发生在类定义时，后续实例会复用这个默认 UUID。列表也用 `default_factory=list` 明确表示“每次创建新列表”。

练习时可以给 `plan.tags` 添加标签，再检查 `another.tags` 是否仍为空。

## 5. 模型、字典和 JSON 怎么来回转换

在 `reading_plan.py` 同目录打开 Python，逐段执行：

```python
from reading_plan import ReadingPlan

plan = ReadingPlan(title="Python 小练习", pages=120)
data = plan.model_dump()
json_ready = plan.model_dump(mode="json")
text = plan.model_dump_json()
restored = ReadingPlan.model_validate_json(text)

print(type(data["plan_id"]).__name__)       # UUID
print(type(json_ready["plan_id"]).__name__) # str
print(type(text).__name__)                  # str
print(restored == plan)                     # True
```

| 目标 | 使用方式 |
| --- | --- |
| 字典输入变模型 | `ReadingPlan.model_validate(data)` |
| JSON 文本输入变模型 | `ReadingPlan.model_validate_json(text)` |
| 模型变 Python 字典 | `plan.model_dump()` |
| 模型变 JSON 兼容字典 | `plan.model_dump(mode="json")` |
| 模型变 JSON 文本 | `plan.model_dump_json()` |
| 生成描述数据结构的规则 | `ReadingPlan.model_json_schema()` |

`model_dump()` 里的 UUID、日期等仍可能是 Python 对象，所以不能假定它总能直接交给 `json.dumps()`。JSON Schema 描述的是结构；`model_dump_json()` 输出的是某条记录。自定义 Python 验证器中的业务逻辑也不一定能完整表达成 JSON Schema。详见 [Serialization](https://docs.pydantic.dev/latest/concepts/serialization/) 和 [JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/)。

## 6. 第一遍可以略读的扩展

| 遇到的需求 | 下一步学习 |
| --- | --- |
| 一个书架包含多个读书计划 | 在另一个模型中声明 `plans: list[ReadingPlan]`，验证会深入嵌套数据 |
| 外部字段叫 `book_title` | `Field(alias="book_title")`；输出别名用 `model_dump(by_alias=True)`，内部属性仍用原字段名 |
| 不想让 `"120"` 自动变成整数 | 对该字段设置 `Field(strict=True)`，或研究模型级严格模式 |
| 校验邮箱格式 | 使用 `EmailStr`，先安装 `"pydantic[email]>=2,<3"`；格式通过不代表邮箱真实存在 |
| 普通函数也要检查参数 | 学习 `@validate_call`；默认不验证返回值，需显式启用 `validate_return=True` |

这些分别可查 [Alias](https://docs.pydantic.dev/latest/concepts/alias/)、[Strict Mode](https://docs.pydantic.dev/latest/concepts/strict_mode/)、[Network Types](https://docs.pydantic.dev/latest/api/networks/) 和 [Validation Decorator](https://docs.pydantic.dev/latest/concepts/validation_decorator/)。

## 7. 常见误区与自测

**创建成功后，随意修改属性仍一定合法吗？** 不一定。默认不会对后续赋值重新校验；有这种需要时研究 `validate_assignment=True`。多字段约束涉及更新顺序时，更容易理解的做法是合并数据后重新 `model_validate()`，验证成功再替换原对象。嵌套列表的原地修改也不能简单等同于属性赋值校验。

**`repr=False` 能保护秘密吗？** 它只是隐藏对象打印时的某个字段，并不自动阻止该字段被序列化。

**看到旧教程中的 `.dict()`、`.json()` 怎么办？** 本笔记使用 Pydantic 2 的 `model_dump()`、`model_dump_json()`；不要随意混用 v1 的 `@validator` 与 v2 的 `@field_validator`。

先预测，再用完整案例验证以下输入：

1. 标题为三个空格，页数为 120，会发生什么？
2. 已读页数等于总页数，会通过吗？
3. 传入额外字段 `rating=5`，会发生什么？
4. 如何规定总页数最多为 2000？

<details markdown="1">
<summary>完成后展开参考答案</summary>

1. 空白被剥离，标题长度为零，字段校验失败。
2. 会。规则只拒绝已读页数大于总页数。
3. `extra="forbid"` 导致 `extra_forbidden` 错误。
4. 把总页数字段改为 `pages: int = Field(gt=0, le=2000)`，分别测试 2000 和 2001。

</details>

## 8. 合上笔记后，试着自己写

把“读书计划”改成“课程学习计划”，包含课程名、总课时和已学课时。从空文件开始，依次完成成功输入、失败输入、JSON 往返三步。能够解释每条规则放在哪里，比背下所有 API 更有用。

下一步阅读 [asyncio 笔记]({% post_url /dev/python/2026-09-09-asyncio-beginner-notes %})，学习如何在等待数据时推进其他任务。
