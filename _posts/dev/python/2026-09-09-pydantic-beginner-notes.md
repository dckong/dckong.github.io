---
title: "Pydantic 学习笔记：从第一个模型到完整的数据入口"
author: gpt6_astra
date: 2026-09-09 08:00:00 +0800
last_modified_at: 2026-09-10
categories: [Dev, Python]
tags: [python, pydantic, beginner, learning-notes]
description: Pydantic 的模型、字段约束、序列化、自定义验证、函数参数校验与配置管理，以读书计划为例说明各项规则的用法和边界。
toc: true
---

Pydantic 是 Python 的数据验证库。它根据类型注解解析和检查输入，把字典、JSON 等外部数据转换成模型对象。字段的类型、取值范围和业务规则可以集中定义在模型中，后续代码直接使用验证后的值。

**运行环境：Python 3.11+，Pydantic 2.11+ 且小于 3；配置管理使用 pydantic-settings 2.x。** 别名示例中的 `validate_by_name` 和 `validate_by_alias` 需要 Pydantic 2.11+。

标注“完整脚本”的示例按文件名保存后即可运行；标注“接着运行”或“替换定义”的片段放入指定脚本。需要互相导入的文件放在同一目录。

## 1. 为什么需要数据验证？

### 1.1 外部数据通常没有准备好直接参加运算

假设一个表单提交了读书计划：

```python
raw = {"title": "Python 数据练习", "pages": "180"}
```

页数看起来是 180，但引号说明它是字符串。执行 `raw["pages"] - 20` 会产生 `TypeError`。减法需要数值，而表单提交的页数仍是文本。

可以用 `int(raw["pages"])` 转换，但还需要处理其他输入情况：字段没传；传了 `"很多"`；传了 `-5`；标题为空；记录嵌套在列表里；其他入口忘记执行转换。逐项写 `if` 可以解决，但转换顺序、错误位置和错误说明都需要手动维护。

这些检查可以集中在数据入口完成：声明允许的类型与取值，解析并验证输入，再将结果交给业务代码。

### 1.2 类型提示不会自动执行运行时检查

完整脚本 `type_hints.py`：

```python
from dataclasses import dataclass


def remaining(pages: int, completed: int) -> int:
    return pages - completed


@dataclass
class PlainPlan:
    title: str
    pages: int


plan = PlainPlan(title="Python 数据练习", pages="180")
print(plan.pages, type(plan.pages).__name__)
try:
    print(remaining(plan.pages, 20))
except TypeError as error:
    print(type(error).__name__)
```

输出：

```text
180 str
TypeError
```

`pages: int` 并没有触发 `int(...)`，标准库 `dataclass` 也没有因为这个注解而验证输入。编辑器和静态检查器可以提醒类型不匹配，但它们与“程序运行时真的收到一条坏数据”不是同一个环节。

Pydantic 会读取模型的注解，在创建模型时执行运行时解析与验证。因此，注解仍然是熟悉的 Python 语法，新增的是理解并执行这些规则的库。模型如何利用注解定义结构，见 [官方 Models 文档](https://docs.pydantic.dev/latest/concepts/models/)。

### 1.3 验证的范围

如果只声明“页数是整数”，`-5` 就没有违反这条规则。若希望页数为正，需要把正数约束写出来。若希望书名真实存在，则需要另外的数据来源去核实；若希望只有作者能修改记录，则属于权限逻辑。

所以，“验证通过”的意思是数据满足**已经声明且确实执行了的规则**。Pydantic 不会猜测业务需求，也不会自动查数据库、认证用户或保证后续任意修改仍然正确。

## 2. 安装：核心库与可选能力分别负责什么？

在自己选定的学习环境里安装核心库：

```bash
python -m pip install "pydantic>=2.11,<3"
```

`python -m pip` 表示使用这个 Python 对应的 pip，可以减少“装在一个环境、运行在另一个环境”的混淆。版本范围限定为 2.11 及以上的 v2 版本。

后面校验邮箱时还需要：

```bash
python -m pip install "pydantic[email]>=2.11,<3"
```

这里的 `[email]` 是安装额外依赖的语法，包含 `EmailStr` 所需的 `email-validator`。引号也能避免 shell 把方括号解释成文件匹配模式。

配置管理需要单独的包：

```bash
python -m pip install "pydantic-settings>=2,<3"
```

安装时名字是 `pydantic-settings`，导入时模块名是 `pydantic_settings`。Pydantic v2 已把 `BaseSettings` 移到这个独立包中。安装入口可查 [Pydantic Installation](https://docs.pydantic.dev/latest/install/)，迁移差异可查 [Migration Guide](https://docs.pydantic.dev/latest/migration/)。

核心模型只需安装 Pydantic。使用 `EmailStr` 时需要 email 额外依赖；普通 `str` 字段不提供邮箱格式校验。

## 3. 定义模型与验证输入

### 3.1 定义和创建模型

完整脚本 `plan_v1.py`：

```python
from pydantic import BaseModel, ValidationError


class PlanV1(BaseModel):
    title: str
    pages: int
    completed_pages: int = 0


if __name__ == "__main__":
    plan = PlanV1(title="Python 数据练习", pages="180")
    print(plan.title)
    print(plan.pages, type(plan.pages).__name__)
    print(plan.completed_pages)
    print(plan.pages - plan.completed_pages)
```

输出：

```text
Python 数据练习
180 int
0
180
```

模型声明中，`PlanV1` 继承 `BaseModel`，获得模型创建、验证和导出能力。`title`、`pages`、`completed_pages` 是字段。没有默认值的前两个字段必须提供；第三个字段省略时使用 0。

调用 `PlanV1(...)` 时，Pydantic 根据这些声明处理输入。默认的宽松模式接受部分可转换的值，所以字符串 `"180"` 被解析成整数 180。此后使用 `plan.pages` 读取的是模型字段，而不是原始字典的值。

转换后的值保存在新建的模型中，业务运算使用的是模型字段。原始输入不会因此被自动改写。转换的接受范围可以通过严格模式控制，见第 6 节。

### 3.2 输入错误与 ValidationError

新建完整脚本 `plan_errors.py`，与 `plan_v1.py` 放在同一目录：

```python
from pydantic import ValidationError
from plan_v1 import PlanV1


try:
    PlanV1.model_validate({"pages": "很多", "completed_pages": "一半"})
except ValidationError as error:
    print("错误数量：", error.error_count())
    for item in error.errors():
        print(item["loc"], item["type"])
```

输出：

```text
错误数量： 3
('title',) missing
('pages',) int_parsing
('completed_pages',) int_parsing
```

这里有三个彼此独立的问题：没有书名、总页数不能解析成整数、已读页数也不能解析成整数。Pydantic 把这些字段错误装进一次 `ValidationError`，让调用方一次看到需要修复的地方。

`error.errors()` 返回错误条目的列表，每项有自己的结构：

| 键 | 回答的问题 | 本例如何阅读 |
| --- | --- | --- |
| `loc` | 错误在哪一层、哪个字段？ | `('pages',)` 表示页数字段 |
| `type` | 错误属于哪类？ | `int_parsing` 表示无法解析整数 |
| `msg` | 人读的说明是什么？ | 适合显示或调试，不应依赖整句永远不变 |
| `input` | 哪个输入触发了错误？ | 可能含用户原始数据，输出前先决定需要哪些字段 |
| `ctx` | 是否有补充上下文？ | 某些错误有，某些没有 |

业务需要的是错误定位时，提取 `loc` 和 `type` 比解析整段英文报错稳定。完整结构见 [官方 Error Handling](https://docs.pydantic.dev/latest/errors/errors/)。

`PlanV1(title="", pages=-5)` **会通过验证**：模型目前只声明了类型，空字符串仍是字符串，负数仍是整数。标题非空和页数为正需要额外的字段约束，见第 5 节。

### 3.3 构造函数、字典和 JSON 是不同的输入入口

新建完整脚本 `plan_inputs.py`，与 `plan_v1.py` 放在同一目录：

```python
from plan_v1 import PlanV1


first = PlanV1(title="类型练习", pages=180)
second = PlanV1.model_validate({"title": "类型练习", "pages": "180"})
third = PlanV1.model_validate_json(
    '{"title": "类型练习", "pages": 180}'
)

print(first == second == third)
print(type(second).__name__)
```

输出 `True` 和 `PlanV1`。三条输入最终得到相同字段值，但入口各有用途：

- 在 Python 代码中明确写出参数时，构造函数直观。
- 已经拿到字典或待验证对象时，`model_validate()` 表达“这里是数据边界”。
- 拿到 JSON 文本或字节时，`model_validate_json()` 同时处理 JSON 解析与模型验证。

JSON 文本与 Python 字典不是同一种对象：前者是字符串，后者是内存中的映射；JSON 的布尔值是 `true/false`，空值是 `null`，Python 对应 `True/False/None`。不要把字典传给要求 JSON 文本的方法。

也不要推断三个入口对所有类型、所有严格模式设置都完全等价。本例只演示简单字段的共同结果；一些类型在 Python 输入模式与 JSON 输入模式下具有不同的接受规则。

### 3.4 必填、允许 None、有默认值，分别由什么决定？

完整脚本 `optional_fields.py`：

```python
from pydantic import BaseModel, ValidationError


class RequiredNote(BaseModel):
    note: str


class NullableNote(BaseModel):
    note: str | None


class DefaultNote(BaseModel):
    note: str | None = None


for model in [RequiredNote, NullableNote, DefaultNote]:
    for data in [{}, {"note": None}, {"note": "稍后阅读"}]:
        try:
            item = model.model_validate(data)
            print(model.__name__, data, "通过", repr(item.note))
        except ValidationError as error:
            print(model.__name__, data, error.errors()[0]["type"])
```

运行之前先预测这张表：

| 声明 | 不传字段 | 传 `None` | 传字符串 |
| --- | --- | --- | --- |
| `note: str` | 缺失错误 | 类型错误 | 通过 |
| <code>note: str &#124; None</code> | 缺失错误 | 通过 | 通过 |
| <code>note: str &#124; None = None</code> | 使用默认值 | 通过 | 通过 |

类型注解决定哪些值能接受；默认值决定能不能省略。`Optional[str]` 等价于 `str | None`，名称里的 Optional 容易让人误解，但在 v2 中它本身并不会提供默认值。这些语义见 [Fields：Default values](https://docs.pydantic.dev/latest/concepts/fields/#default-values)。

## 4. 模型的出口：Python 字典、JSON 与 Schema

### 4.1 日期与枚举类型

如果所有字段都只有字符串和整数，容易误以为导出字典与导出 JSON 只是换一种打印格式。加入日期和枚举后，区别就明显了。

完整脚本 `plan_io.py`：

```python
from datetime import date
from enum import Enum
from pydantic import BaseModel


class ReadingState(str, Enum):
    PLANNED = "planned"
    READING = "reading"
    FINISHED = "finished"


class DatedPlan(BaseModel):
    title: str
    pages: int
    starts_on: date
    state: ReadingState = ReadingState.PLANNED


if __name__ == "__main__":
    plan = DatedPlan.model_validate({
        "title": "异步基础",
        "pages": "120",
        "starts_on": "2026-09-10",
        "state": "reading",
    })
    print(type(plan.starts_on).__name__)
    print(plan.starts_on.year)
    print(plan.state.value)

    python_data = plan.model_dump()
    json_data = plan.model_dump(mode="json")
    json_text = plan.model_dump_json()
    print(type(python_data["starts_on"]).__name__)
    print(type(json_data["starts_on"]).__name__)
    print(type(json_text).__name__)
    print(DatedPlan.model_validate_json(json_text) == plan)
```

输出：

```text
date
2026
reading
date
str
str
True
```

`date` 是 Python 的日期类型。把字符串解析成日期后，程序就能访问 `.year` 或做日期运算，并且 `"2026-02-30"` 这样的不存在的日期会失败。枚举则把状态限定为三个选项；输入 `"almost_done"` 无法对应其中任何一个成员。

### 4.2 三种导出格式

`model_dump()` 为 Python 程序提供字典，日期等值可以保留为 Python 对象。`model_dump(mode="json")` 仍然返回字典，但字段值转成 JSON 能表达的形式。`model_dump_json()` 直接返回 JSON 字符串。

```text
外部字典 ── model_validate ──> 模型实例
JSON 文本 ── model_validate_json ──> 模型实例

模型实例 ── model_dump ──> Python 字典
模型实例 ── model_dump(mode="json") ──> JSON 兼容字典
模型实例 ── model_dump_json ──> JSON 文本
```

如果把包含 `date` 的普通导出结果直接交给 `json.dumps()`，标准 JSON 编码器不知道如何处理该日期对象，就可能报错。反过来，对 `model_dump_json()` 的结果再次 `json.dumps()`，得到的是“一个字符串的 JSON 表示”，不是原先那条记录的 JSON 对象。

一个判断方法是先看 `type(...)`：下一层接口要字典，就给字典；要 JSON 文本，就给文本。序列化模式见 [官方 Serialization](https://docs.pydantic.dev/latest/concepts/serialization/)。

### 4.3 JSON Schema 描述规则，不是一条数据

接着在 `plan_io.py` 后面运行：

```python
schema = DatedPlan.model_json_schema()
print(schema["required"])
print(schema["properties"]["starts_on"])
print(schema["$defs"]["ReadingState"]["enum"])
```

这里应该看到必填字段包含 `title`、`pages`、`starts_on`；日期字段带有 `format: date`；状态定义包含三个枚举值。`state` 有默认值，因此不在必填列表里。

JSON 记录保存具体的字段值，Schema 描述记录应具有的结构与约束。接口文档生成器可以利用 Schema 展示字段、类型、必填性和说明，其他工具也可以用它构建校验或表单。

不过，第 7 节中用任意 Python 代码写出的跨字段规则，不会因此自动获得等价的跨语言实现。Schema 可表达的约束与 Python 函数的表达能力不同；不同 Schema 校验器对 `format` 的执行方式也需要单独确认。见 [官方 JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/)。

## 5. 使用 Field：逐条把业务要求写进字段

### 5.1 类型够了，范围还不够

读书计划新增三个要求：标题长度为 1–80；总页数为正且不超过 5000；已读页数不能是负数。

完整脚本 `plan_v2.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class PlanV2(BaseModel):
    title: str = Field(min_length=1, max_length=80)
    pages: int = Field(gt=0, le=5000)
    completed_pages: int = Field(default=0, ge=0)


if __name__ == "__main__":
    print(PlanV2(title="数据校验", pages="180"))
    for data in [
        {"title": "", "pages": 180},
        {"title": "数据校验", "pages": 0},
        {"title": "数据校验", "pages": 5001},
        {"title": "数据校验", "pages": 180, "completed_pages": -1},
    ]:
        try:
            PlanV2.model_validate(data)
        except ValidationError as error:
            print(error.errors()[0]["loc"], error.errors()[0]["type"])
```

四个错误依次是 `string_too_short`、`greater_than`、`less_than_equal`、`greater_than_equal`。

`Field(...)` 用来附加字段规则与元数据。`title: str = Field(min_length=1)` **仍是必填字段**：等号右边是字段声明信息，并没有提供书名默认值。要设置默认值，需像 `completed_pages` 一样显式给出 `default=0`。

| 写法 | 数学含义 | 自己应试的两个边界 |
| --- | --- | --- |
| `gt=0` | 大于 0 | 0 与 1 |
| `ge=0` | 大于等于 0 | -1 与 0 |
| `lt=5` | 小于 5 | 4 与 5 |
| `le=5` | 小于等于 5 | 5 与 6 |
| `min_length=1` | 长度至少为 1 | 空串与一个字符 |

现在试 `title="   "`：仍能通过，因为三个空格的长度为 3。再试 `pages=100, completed_pages=101`：也能通过，因为两个字段分别满足正数和非负数约束。前者需要规范化输入，后者需要检查字段之间的关系；第 7 节分别处理。

### 5.2 默认工厂：每次创建的值，不能提前只计算一次

计划需要一个 ID。完整脚本 `default_factory.py`：

```python
from uuid import UUID, uuid4
from pydantic import BaseModel, Field


class FixedDefault(BaseModel):
    plan_id: UUID = uuid4()


class FreshDefault(BaseModel):
    plan_id: UUID = Field(default_factory=uuid4)
    tags: list[str] = Field(default_factory=list)


print(FixedDefault().plan_id == FixedDefault().plan_id)
first = FreshDefault()
second = FreshDefault()
print(first.plan_id == second.plan_id)
first.tags.append("python")
print(first.tags, second.tags)
```

输出：

```text
True
False
['python'] []
```

第一种写法在**执行类定义**时调用 `uuid4()`，得到一个值放进默认项。后面省略字段时复用该值，自然不会为每个实例重新生成 ID。

`default_factory=uuid4` 交给模型的是函数本身；每次创建实例且没有传入 `plan_id` 时才调用它。`default_factory=list` 同理表达“这个实例需要一个新列表”。不要写 `default_factory=uuid4()`，那会把 UUID 值当成应该调用的函数。

Pydantic 对不可哈希的可变默认值有自己的深拷贝处理，因此不能照搬普通函数默认列表的结论，说所有模型中的 `tags=[]` 都必然共享。但显式工厂更容易表达意图。默认值、工厂和验证行为见 [Fields 文档](https://docs.pydantic.dev/latest/concepts/fields/)。

### 5.3 默认值与显式输入，默认不走完全相同的验证路径

完整脚本 `validate_defaults.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class UncheckedDefault(BaseModel):
    pages: int = "unknown"


class CheckedDefault(BaseModel):
    pages: int = Field(default="unknown", validate_default=True)


print(UncheckedDefault().pages)
try:
    CheckedDefault()
except ValidationError as error:
    print(error.errors()[0]["type"])
```

输出 `unknown` 与 `int_parsing`。第一个默认值故意写错，用来暴露普通模型默认值通常不验证这一点。第二个通过 `validate_default=True` 把默认值也纳入验证。

这意味着：希望某个字段验证器处理默认值时，也需要考虑是否启用了默认值验证。`BaseSettings` 的默认值行为有所不同，不能把普通 `BaseModel` 的结论无条件套过去。

### 5.4 字段别名：输入名、属性名与输出名

外部表单把书名称作 `bookTitle`，Python 内部希望使用 `title`。完整脚本 `field_alias.py`：

```python
from pydantic import BaseModel, ConfigDict, Field, ValidationError


class ExternalPlan(BaseModel):
    title: str = Field(alias="bookTitle")
    pages: int


plan = ExternalPlan.model_validate({"bookTitle": "数据校验", "pages": 180})
print(plan.title)
print(plan.model_dump())
print(plan.model_dump(by_alias=True))

try:
    ExternalPlan(title="数据校验", pages=180)
except ValidationError as error:
    print(error.errors()[0]["loc"], error.errors()[0]["type"])


class BothNamesPlan(ExternalPlan):
    model_config = ConfigDict(validate_by_name=True, validate_by_alias=True)


print(BothNamesPlan(title="数据校验", pages=180).title)
```

前两次导出分别使用 `title` 与 `bookTitle`，失败项是 `('bookTitle',) missing`。在本例默认配置下，声明别名后输入按别名接收，**不是自动同时接受两个名字**。内部访问始终是 `plan.title`。

最后的类显式允许别名与字段名两种输入。这里的两个配置项需要 Pydantic 2.11+，较早的 v2 教程可能使用 `populate_by_name=True`。

如果输入叫 `bookTitle`、输出却必须叫 `book_title`，可把字段声明改成：

```python
# 替换 ExternalPlan 类里的 title 声明
    title: str = Field(validation_alias="bookTitle", serialization_alias="book_title")
```

粘贴时保持类内的正常缩进。`validation_alias` 负责输入命名，`serialization_alias` 负责输出命名；输出别名仍需 `by_alias=True`。不要把模型内部的属性名也一起改掉。细节见 [官方 Alias](https://docs.pydantic.dev/latest/concepts/alias/)。

### 5.5 冻结、隐藏打印、排除导出，分别在哪个环节生效？

完整脚本 `field_visibility.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class ImportRecord(BaseModel):
    record_id: int = Field(frozen=True)
    owner_note: str = Field(repr=False)
    temporary_marker: str = Field(exclude=True)


record = ImportRecord(record_id=1, owner_note="仅自己可见", temporary_marker="batch-a")
print(record)
print(record.model_dump())
try:
    record.record_id = 2
except ValidationError as error:
    print(error.errors()[0]["type"])
```

打印模型时没有 `owner_note`，导出字典时却有它；`temporary_marker` 在模型打印里可以出现，常规导出时被排除；给 `record_id` 重新赋值产生 `frozen_field`。

因此 `repr=False` 不能当成保密机制，`exclude=True` 不能当成禁止访问，`frozen=True` 也不表示内部嵌套列表变成不可变集合。如果字段是列表，禁止重新赋值和禁止 `append()` 是两件事。

元数据同样要区分职责：`Field(description="总页数必须为正")` 只是描述；真正实现正数规则的是 `gt=0`。把这句话写进描述，可以让文档清楚，但不能替代约束。

### 5.6 格式约束与文档元数据

如果书籍编号必须形如 `BK-001`，可以用 `pattern` 描述格式。完整脚本 `field_pattern.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class BookCode(BaseModel):
    code: str = Field(
        pattern=r"^BK-[0-9]{3}$",
        title="书籍编号",
        description="BK- 前缀后接三位数字，例如 BK-001",
        examples=["BK-001"],
    )


for value in ["BK-001", "BK-1", "xBK-001", "BK-001-extra"]:
    try:
        print(BookCode(code=value).code)
    except ValidationError as error:
        print(error.errors()[0]["type"])

print(BookCode.model_json_schema()["properties"]["code"])
```

只有第一项通过，其他项是 `string_pattern_mismatch`。正则中的 `^` 和 `$` 指定边界，`[0-9]{3}` 表示三位数字；如果只给一个没有边界的片段，匹配到片段不等于整个编号符合要求。复杂正则还要留意 Pydantic 默认正则引擎支持的语法，不要假定与 Python `re` 的所有特性完全相同。

最后打印的 Schema 属性同时包含格式规则、标题、描述和示例。格式规则执行验证，后面三项帮助工具和读者理解这个字段。把 examples 改成另一个编号不会改变输入接受范围；把 description 写成“编号必须唯一”也不会自动查重。

生成的 Schema 包含字段类型、格式约束和元数据，可供接口文档与其他工具使用。

## 6. 模型变复杂后：控制转换、额外字段与嵌套数据

### 6.1 宽松模式与严格模式

完整脚本 `strict_pages.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class FlexiblePages(BaseModel):
    pages: int


class StrictPages(BaseModel):
    pages: int = Field(strict=True)


for value in [100, "100", 100.0, 100.5, True]:
    for model in [FlexiblePages, StrictPages]:
        try:
            result = model(pages=value)
            print(model.__name__, repr(value), "通过", result.pages)
        except ValidationError:
            print(model.__name__, repr(value), "拒绝")
```

对本例整数类型，可以用下面的结果检查预测：

| 输入 | 宽松模式 | 严格整数 |
| --- | --- | --- |
| `100` | 100 | 100 |
| `"100"` | 100 | 拒绝 |
| `100.0` | 100 | 拒绝 |
| `100.5` | 拒绝，不会随意截断小数 | 拒绝 |
| `True` | 1 | 拒绝 |

表单或 CSV 常给出数字文本，宽松模式可以减少调用方的重复转换。内部协议要求真正的整数时，严格模式能更早发现类型错误。

还可以为整个模型设置 `ConfigDict(strict=True)`，或在单次入口使用 `FlexiblePages.model_validate(data, strict=True)`。但不要把严格整数实验推广为“任何类型、任何入口都禁止转换”。JSON 没有原生日期对象，日期等类型在严格 JSON 模式下仍可能接受规范字符串。见 [Strict Mode](https://docs.pydantic.dev/latest/concepts/strict_mode/) 与 [Conversion Table](https://docs.pydantic.dev/latest/concepts/conversion_table/)。

### 6.2 多出的字段：忽略、拒绝、保留是不同策略

默认情况下，未声明字段通常被忽略。假设调用方把 `completed_pages` 拼成 `completed_page`，模型可能忽略这个拼错的字段，再使用已读页数默认值 0。程序没有报错，但业务含义已经变化。

完整脚本 `extra_fields.py`：

```python
from pydantic import BaseModel, ConfigDict, ValidationError


class ImportPlan(BaseModel):
    model_config = ConfigDict(extra="forbid")
    title: str
    pages: int
    completed_pages: int = 0


try:
    ImportPlan(title="数据校验", pages=180, completed_page=20)
except ValidationError as error:
    print(error.errors()[0]["loc"], error.errors()[0]["type"])
```

输出 `('completed_page',) extra_forbidden`。`extra="ignore"` 是忽略，`extra="forbid"` 是拒绝，`extra="allow"` 是保留。选择哪种取决于入口约定：严格的导入模板常希望发现拼写错误，兼容持续扩展的上游数据则可能允许未知字段。

`ConfigDict` 是模型级配置，`Field` 是字段级规则；两者不要混为一谈。见 [Models：Extra data](https://docs.pydantic.dev/latest/concepts/models/#extra-data)。

### 6.3 嵌套模型：验证会沿着数据结构继续往里走

现在一份书单包含多本书。完整脚本 `nested_shelf.py`：

```python
from pydantic import BaseModel, Field, ValidationError


class Book(BaseModel):
    title: str
    pages: int = Field(gt=0)


class Shelf(BaseModel):
    name: str
    books: list[Book] = Field(min_length=1)


shelf = Shelf.model_validate({
    "name": "九月学习",
    "books": [
        {"title": "数据校验", "pages": "180"},
        {"title": "异步基础", "pages": 120},
    ],
})
print(type(shelf.books[0]).__name__)
print(shelf.books[0].pages)
print(shelf.model_dump())

try:
    Shelf.model_validate({
        "name": "九月学习",
        "books": [{"title": "数据校验", "pages": 180},
                  {"title": "异步基础", "pages": 0}],
    })
except ValidationError as error:
    print(error.errors()[0]["loc"])
```

第一项类型是 `Book`，页数是整数 180；导出时内层模型也变成字典。失败位置是 `('books', 1, 'pages')`：进入书籍列表，找到索引 1 的第二项，再看它的页数。

`list` 只说明是列表；`list[Book]` 才说明每个元素按 Book 验证。列表本身非空与元素字段合法也分别由外层 `min_length=1` 和内层规则负责。

一个外层模型的配置不会自动成为所有内层模型的统一配置。如果每本书也要拒绝未知字段，就在 `Book` 自己的配置中声明。结构与错误路径一起理解后，之后即使嵌套更深，也能沿着 `loc` 定位。

## 7. 自定义验证：类型和 Field 表达不完的部分

### 7.1 字段验证器：检查标题

标题除满足长度约束外，还需要去掉首尾空白，并拒绝“待定”这样的占位词。完整脚本 `title_validator.py`：

```python
from pydantic import BaseModel, ConfigDict, Field, ValidationError, field_validator


class NamedPlan(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)
    title: str = Field(min_length=1)

    @field_validator("title", mode="after")
    @classmethod
    def reject_placeholder(cls, value: str) -> str:
        if value == "待定":
            raise ValueError("请填写具体书名")
        return value


for title in ["  数据校验  ", "   ", " 待定 "]:
    try:
        print(NamedPlan(title=title).title)
    except ValidationError as error:
        print(error.errors()[0]["type"])
```

输出为 `数据校验`、`string_too_short`、`value_error`。

验证按以下顺序执行：输入先按字符串规则处理，并剥离两端空白；内建长度检查拒绝空串；通过后，`mode="after"` 的字段验证器得到字符串；自定义规则再拒绝占位词。第二项在长度检查时已经失败，不需要进入后面的成功路径。

`@field_validator("title")` 把方法登记到指定字段。`@classmethod` 表明它是类方法，首参数 `cls` 是模型类，此时还不需要一个已经完整创建好的实例。

**成功时必须返回要保留的值。** 没有 `return value` 的函数会返回 `None`；after 验证器的返回值不会神奇地再完整通过同一套字段类型检查，所以忘写返回值会破坏你以为已经建立的类型保证。

校验失败用 `ValueError` 表达，Pydantic 将其纳入 `ValidationError`。不要用可能被优化选项禁用的 `assert` 实现必要业务检查，也不要把编程错误产生的 `TypeError` 当成必然会被包装的数据错误。见 [官方 Validators](https://docs.pydantic.dev/latest/concepts/validators/)。

### 7.2 before 与 after：拿到的值处于不同阶段

导入模板可能把页数写成 `" 180 页 "`。整数解析器本来不接受中文单位，可以先移除明确允许的后缀，再让正常整数校验继续负责。

完整脚本 `validator_pipeline.py`：

```python
from typing import Any
from pydantic import BaseModel, Field, ValidationError, field_validator


class ImportedPages(BaseModel):
    pages: int = Field(gt=0)

    @field_validator("pages", mode="before")
    @classmethod
    def remove_unit(cls, value: Any) -> Any:
        print("before:", repr(value), type(value).__name__)
        if isinstance(value, str):
            return value.strip().removesuffix("页").strip()
        return value

    @field_validator("pages", mode="after")
    @classmethod
    def check_upper_bound(cls, value: int) -> int:
        print("after:", repr(value), type(value).__name__)
        if value > 5000:
            raise ValueError("页数过大，请核对来源")
        return value


for raw in [" 180 页 ", "很多页", -1, 6000]:
    try:
        print("结果:", ImportedPages(pages=raw).pages)
    except ValidationError as error:
        print("失败:", error.errors()[0]["type"])
```

对第一项，before 看到原始字符串，返回 `"180"`；内建规则把它解析为正整数；after 得到 `180 int`，最后模型保存 180。

对 `"很多页"`，before 返回 `"很多"`，整数解析失败，因此没有 after 日志。对 -1，类型解析成功，但正数约束失败，也不会进入 after。对 6000，前面都成功，最后由自定义规则拒绝。

```text
原始值 → before → 类型解析与 Field 约束 → after → 字段值
```

before 参数标成 `Any`，因为输入也可能是整数、列表或其他对象；必须先判断类型，再调用字符串方法。只因为希望收到字符串，就直接写 `value.strip()`，反而会把坏输入变成验证器自己的程序错误。

这里把上限放进 after 是为了观察执行顺序。实际只需要数值上限时，`Field(le=5000)` 更清楚。验证器应该填补内建规则表达不了的部分。

### 7.3 plain、wrap 与验证顺序

它们并不是 before/after 的更高级替代名。`plain` 会截断通常的内部验证流程，如果返回了不合类型的值，也可能被当作字段结果。`wrap` 会拿到一个 handler，允许你在调用正常验证前后插入处理，甚至选择不调用它。

清理页数后缀可以用 before，检查解析后的值可以用 after。需要捕获内部校验错误并按错误类型处理输入时，可以使用 wrap；如果跳过 handler，正常的类型与约束检查也会被跳过。

多个验证器组合还存在顺序规则。特别是 `Annotated` 中 before/wrap 从右向左、after 从左向右执行；不要把一个简单实验的顺序推广到所有组合。可查 [Validators：Ordering of validators](https://docs.pydantic.dev/latest/concepts/validators/#ordering-of-validators)。

### 7.4 多个字段之间的规则：已读页数不能超过总页数

完整脚本 `progress_validator.py`：

```python
from typing import Self
from pydantic import BaseModel, Field, ValidationError, model_validator


class ProgressPlan(BaseModel):
    pages: int = Field(gt=0)
    completed_pages: int = Field(default=0, ge=0)

    @model_validator(mode="after")
    def check_progress(self) -> Self:
        if self.completed_pages > self.pages:
            raise ValueError("已读页数不能超过总页数")
        return self


for completed in [0, 100, 101]:
    try:
        plan = ProgressPlan(pages=100, completed_pages=completed)
        print("剩余", plan.pages - plan.completed_pages)
    except ValidationError as error:
        print(error.errors()[0]["loc"], error.errors()[0]["msg"])
```

输出剩余 100、剩余 0，最后是模型级错误 `()` 与“已读页数不能超过总页数”。

模型验证器拿到的是 `self`，即字段验证通过后的实例，所以可以同时比较两个字段。`Self` 是返回当前类实例的类型注解，Python 3.11 起可从 `typing` 导入。成功要返回 `self`。

如果把 `pages` 改为 `"很多"`，字段本身先失败，after 模型验证器不会拿着一个正常实例继续比较。先检查单字段能否成立，再检查字段关系，这就是两层验证的分工。

字段验证器也可以通过 `ValidationInfo.data` 访问先前已经验证的其他字段，但会受到声明顺序和前面字段是否成功的影响。跨字段不变量放进 after 模型验证器，通常更容易读，也避免依赖偶然的字段顺序。

### 7.5 检查开始日期与结束日期

完整脚本 `reading_window.py`：

```python
from datetime import date
from typing import Self
from pydantic import BaseModel, ValidationError, model_validator


class ReadingWindow(BaseModel):
    starts_on: date
    ends_on: date

    @model_validator(mode="after")
    def dates_in_order(self) -> Self:
        if self.ends_on < self.starts_on:
            raise ValueError("结束日期不能早于开始日期")
        return self


for end in ["2026-09-10", "2026-09-01", "2026-08-31", "2026-02-30"]:
    try:
        window = ReadingWindow(starts_on="2026-09-01", ends_on=end)
        print((window.ends_on - window.starts_on).days)
    except ValidationError as error:
        print(error.errors()[0]["loc"])
```

输出 9、0、`()`、`('ends_on',)`。倒数第二项是日期关系错误，最后一项是日期本身不合法，这两种错误不应该混在一起解释。

固定日期让今天和下个月运行都能复现相同结果。若实际需求涉及年龄，不要用 `365 * 年数` 代替生日判断，也不要直接把今天的年份减去若干年构造日期而忽略 2 月 29 日。先明确“到哪一天算满一岁”这类业务边界，再实现并测试；日期能被解析只是第一层。

### 7.6 邮箱类型与自定义域名规则，是两层不同检查

安装前文的 email 额外依赖后，运行完整脚本 `reader_email.py`：

```python
from pydantic import BaseModel, EmailStr, ValidationError, field_validator


class Reader(BaseModel):
    email: EmailStr

    @field_validator("email", mode="after")
    @classmethod
    def require_study_domain(cls, value: str) -> str:
        domain = value.rsplit("@", 1)[1].lower()
        if domain != "example.com":
            raise ValueError("学习小组只接受 example.com 邮箱")
        return value


for address in ["learner@example.com", "not-an-email", "learner@example.org"]:
    try:
        print(Reader(email=address).email)
    except ValidationError as error:
        print(error.errors()[0]["loc"], error.errors()[0]["msg"])
```

第一项通过。第二项没有通过邮箱类型检查，第三项邮箱格式可以成立，但不满足小组的域名限制。域名检查放在 after，是因为它依赖前面已经确认这是一个有效邮箱形式。

`EmailStr` 并不会替你发送验证邮件或证明收件箱归当前用户所有。格式规范、域名业务规则、账号所有权是不同问题。网络相关类型的行为见 [Network Types](https://docs.pydantic.dev/latest/api/networks/)。

### 7.7 完整模型与更新验证

下面的模型包含独立 ID、标题清理、页数约束和阅读进度检查。完整脚本 `reading_plan.py`：

```python
from typing import Self
from uuid import UUID, uuid4
from pydantic import BaseModel, ConfigDict, Field, field_validator, model_validator


class ReadingPlan(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    plan_id: UUID = Field(default_factory=uuid4, frozen=True)
    title: str = Field(min_length=1, max_length=80)
    pages: int = Field(gt=0, le=5000)
    completed_pages: int = Field(default=0, ge=0)
    tags: list[str] = Field(default_factory=list)
    note: str | None = None

    @field_validator("title", mode="after")
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
    plan = ReadingPlan(title="  数据校验  ", pages="180", completed_pages=30)
    print(plan.title)
    print("剩余页数：", plan.pages - plan.completed_pages)
    print(ReadingPlan.model_validate_json(plan.model_dump_json()) == plan)
```

创建成功不意味着以后赋值自动校验。普通模型默认允许非冻结字段被直接赋上新值；需要重新校验赋值时，可配置 `validate_assignment=True`。但容器的原地修改不等于属性赋值，多个字段依次更新也可能经过临时不一致的状态。

对本例多字段更新，先构造候选数据并完整验证，成功后再替换对象，会更容易推理。完整脚本 `update_plan.py`，与 `reading_plan.py` 放在同一目录：

```python
from pydantic import ValidationError
from reading_plan import ReadingPlan

original = ReadingPlan(title="数据校验", pages=180, completed_pages=30)
candidate = {**original.model_dump(), "completed_pages": 181}
try:
    replacement = ReadingPlan.model_validate(candidate)
except ValidationError:
    print("更新被拒绝，原计划仍是", original.completed_pages)
```

输出原计划仍是 30。这里没有先修改原实例，因此失败后不需要猜测旧对象是否被部分改变。

不要把 `model_copy(update=...)` 当成“验证后更新”：其中的更新数据不会自动完整验证。`model_construct()` 也会绕过正常验证；二者都不适合拿来代替不可信数据入口。相关行为见 [BaseModel API](https://docs.pydantic.dev/latest/api/base_model/)。

## 8. 验证普通函数：把数据规则放到调用边界

### 8.1 使用 validate_call 验证参数

读书计划需要保存和导出，使用模型合适。但“根据每天学习时间计算每周总分钟数”是一个函数调用，参数验证可以直接写在函数入口。

完整脚本 `weekly_minutes.py`：

```python
from typing import Annotated
from pydantic import Field, ValidationError, validate_call


@validate_call
def weekly_minutes(
    minutes_per_day: Annotated[int, Field(gt=0, le=240)],
    days_per_week: Annotated[int, Field(ge=1, le=7)],
) -> int:
    print("进入函数体")
    return minutes_per_day * days_per_week


print(weekly_minutes("30", 5))
try:
    weekly_minutes(30, 8)
except ValidationError as error:
    print(error.errors()[0]["loc"], error.errors()[0]["type"])
```

第一次先输出“进入函数体”，再输出 150；第二次直接得到 `(1,) less_than_equal`，不会再次打印“进入函数体”。说明校验发生在函数业务逻辑之前。

`Annotated[int, Field(...)]` 可以拆成两部分：基础类型 `int`，以及附加给 Pydantic 的约束元数据。它本身不会使所有 Python 函数自动验证；这里由 `@validate_call` 读取并执行。

错误位置 `(1,)` 表示第二个位置参数；改成 `weekly_minutes(minutes_per_day=30, days_per_week=8)` 时，错误可以按关键字参数名定位。默认宽松转换仍在工作，所以字符串 `"30"` 被转换成整数后才交给函数体。

### 8.2 返回值注解默认也不会自动变成运行时保证

完整脚本 `validated_return.py`：

```python
from pydantic import ValidationError, validate_call


@validate_call
def unchecked() -> int:
    return "wrong"


@validate_call(validate_return=True)
def checked() -> int:
    return "wrong"


print(unchecked(), type(unchecked()).__name__)
try:
    checked()
except ValidationError:
    print("返回值不符合规则")
```

默认返回 `wrong str`；显式开启返回值验证后才失败。注意返回值验证也可能发生转换，如果返回的是 `"123"`，宽松整数规则可以将其变成 123，而不是必然报错。

如果函数已经写文件、发起请求或修改数据库，返回值校验失败不会自动撤回那些副作用。参数验证负责调用前的数据边界，事务和重试仍由应用设计。规则与选项见 [Validation Decorator](https://docs.pydantic.dev/latest/concepts/validation_decorator/)。

## 9. 管理配置：让程序启动时就发现配置错误

### 9.1 环境变量中的数字通常先是文本

本地运行希望并发数是 2，服务器可能是 8。这些运行参数适合由部署环境提供。若直接使用 `os.environ.get("STUDY_CONCURRENCY")`，得到的通常是字符串或 `None`，仍需处理类型转换、默认值和取值范围。

`BaseSettings` 把不同来源的配置收集起来，再应用模型规则。先不考虑 `.env`，只看一个小模型。安装配置包后，运行完整脚本 `settings_basic.py`：

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class StudySettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="STUDY_")
    daily_minutes: int = Field(default=30, ge=1, le=240)
    concurrency: int = Field(default=2, ge=1, le=10)
    debug: bool = False


if __name__ == "__main__":
    settings = StudySettings()
    print(settings.model_dump())
```

没有对应环境变量时，输出 30、2、False 这组默认值。在 macOS/Linux 终端里只为一次运行提供变量：

```bash
STUDY_DAILY_MINUTES=45 STUDY_DEBUG=true python settings_basic.py
```

输出中的分钟数是整数 45，debug 是布尔值 True。`env_prefix="STUDY_"` 把字段名对应到带前缀的环境变量，默认匹配不区分大小写。

接着分别改成 `STUDY_DAILY_MINUTES=oops` 和 `STUDY_CONCURRENCY=0`。前者无法解析，后者类型能成立但不满足范围。失败发生在创建 settings 对象时，程序还没有开始用坏配置安排工作。

Windows PowerShell 可以先用 `$env:STUDY_DAILY_MINUTES="45"` 设置，再运行脚本；实验结束后删除自己设置的变量，避免影响后续结果。进程运行后外部修改配置不会自动让已创建的对象更新；需要重新加载或重新实例化。

### 9.2 为什么还需要 .env？它从哪里被读取？

变量多了以后，每次在命令前写一串内容不方便。新建完整脚本 `settings_file.py`，并把下方 `.env` 放在同一学习目录：

```python
from pathlib import Path
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class FileSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="STUDY_",
        env_file=Path(__file__).with_name(".env"),
        env_file_encoding="utf-8",
        extra="forbid",
    )
    daily_minutes: int = Field(default=30, ge=1, le=240)
    concurrency: int = Field(default=2, ge=1, le=10)
    debug: bool = False


if __name__ == "__main__":
    print(FileSettings().model_dump())
```

`.env` 内容：

```text
STUDY_DAILY_MINUTES=40
STUDY_CONCURRENCY=3
STUDY_DEBUG=false
```

这个例子用 `Path(__file__).with_name(".env")` 指向脚本旁边的文件。若只写 `env_file=".env"`，相对路径按照**启动命令时的工作目录**解析，不会自动在所有父目录里搜索。明确路径，可以避免在编辑器里运行成功、换目录运行却读不到的困惑。

这里 `.env` 只是配置来源之一，不需要先把里面的值导入 shell。`pydantic-settings` 会读取文件并为模型准备输入；它也不会因为读了 `.env` 就把当前进程所有环境变量改写成文件内容。

### 9.3 不同地方给了不同的值，谁覆盖谁？

采用默认来源顺序，未启用 CLI 来源或自定义设置源时，常见优先级为：

```text
构造函数显式参数 > 进程环境变量 > .env > secrets 目录 > 字段默认值
```

在 `settings_file.py` 后面加一行显式传入参数的调用，对比配置来源的优先级：

```python
print("显式传入：", FileSettings(daily_minutes=50).daily_minutes)
```

按顺序观察：

1. 没有相关环境变量：正常实例读取 `.env` 中的 40。
2. 用 `STUDY_DAILY_MINUTES=45 python settings_file.py`：正常实例得到 45。
3. 同一次执行里，显式传入的实例仍得到 50。
4. 把显式参数改成 0：验证失败，不会因为 `.env` 有合法的 40 就自动退回 40。

来源优先级决定选哪个输入，验证决定该输入是否允许。二者是连续步骤，不是“高优先级错了就悄悄找低优先级补救”。这组来源规则见 [Settings Management：Field value priority](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#field-value-priority)。

### 9.4 大小写与多余配置

在当前 `.env` 中加入 `STUDY_UNKNOWN=1`，`extra="forbid"` 会把未知项报告为错误。这并不表示系统环境里不能有 `PATH` 等其他变量；环境来源只提取相关变量，而 dotenv 来源对额外项有自己的处理方式。

若设置 `case_sensitive=True`，本例字段 `daily_minutes` 与前缀 `STUDY_` 对应的精确名称是 `STUDY_daily_minutes`。不要一边开启区分大小写，一边继续假定全大写名称总能匹配。Windows 环境变量的大小写行为还受平台限制。

同时需要理解模型名与环境变量名的差别：`FileSettings(daily_minutes=50)` 的参数仍是 Python 字段名，不是 `STUDY_DAILY_MINUTES`。别名与前缀组合时也应重新确认命名约定，不能简单把前缀贴到任何 alias 前面。

`.env` 中若包含真实密钥，应该从公开仓库中排除，使用无真实凭据的 `.env.example` 描述需要的项目。`SecretStr` 能遮罩常规显示，但不是加密。普通模型和 Settings 对默认值的验证默认也不同：Settings 默认验证默认值。进一步选项见 [Settings Management](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)。

## 10. 综合实践：导入一批计划，明确选择失败策略

### 10.1 逐条接受：一条坏数据不丢掉其他好数据

与 `reading_plan.py`、`settings_basic.py` 放在同一目录，运行完整脚本 `import_plans.py`：

```python
from pydantic import ValidationError
from reading_plan import ReadingPlan
from settings_basic import StudySettings


def import_plans(rows: list[dict]) -> tuple[list[ReadingPlan], list[dict]]:
    accepted = []
    rejected = []
    for row_number, row in enumerate(rows, start=1):
        try:
            accepted.append(ReadingPlan.model_validate(row))
        except ValidationError as error:
            rejected.append({
                "row": row_number,
                "problems": [
                    {"field": list(item["loc"]), "type": item["type"]}
                    for item in error.errors()
                ],
            })
    return accepted, rejected


if __name__ == "__main__":
    settings = StudySettings(daily_minutes=45)
    rows = [
        {"title": " 数据校验 ", "pages": "180", "completed_pages": 30},
        {"title": "异步基础", "pages": 120},
        {"title": "   ", "pages": -1},
        {"title": "函数练习", "pages": 100, "completed_pages": 101},
    ]
    plans, errors = import_plans(rows)
    print("每天学习分钟数：", settings.daily_minutes)
    print("成功：", len(plans), "失败：", len(errors))
    for plan in plans:
        print(plan.title, "剩余", plan.pages - plan.completed_pages)
    print(errors)
```

预期成功 2 条，失败 2 条；剩余页数分别是 150 和 120。第 3 行有两个字段错误，第 4 行有一个模型关系错误，因此“失败行数”与“错误条目数”不相等。

这里的流程可以逐步追踪：先验证程序配置；逐行验证业务记录；只对成功模型做业务运算；把失败位置整理为上层能展示的结构。配置的每日学习分钟数没有被当作记录的页数，两类数据的意义清楚地分开了。

错误输出只取需要的字段，不直接把整个 `error.errors()` 交给 `json.dumps()`，因为上下文可能含异常对象等不能直接编码的值，也可能包含没必要展示的原始输入。

### 10.2 使用 TypeAdapter 验证整批数据

完整脚本 `whole_batch.py`，放在 `reading_plan.py` 同目录：

```python
from pydantic import TypeAdapter, ValidationError
from reading_plan import ReadingPlan


adapter = TypeAdapter(list[ReadingPlan])
try:
    plans = adapter.validate_python([
        {"title": "数据校验", "pages": 180},
        {"title": "异步基础", "pages": 0},
    ])
except ValidationError as error:
    print(error.errors()[0]["loc"])
```

输出 `(1, 'pages')`，表示整个列表中第二条记录的页数字段失败。`TypeAdapter` 让 `list[ReadingPlan]` 这样的类型表达式直接成为验证目标，不必为了列表再造一个只有单字段的模型。见 [官方 Type Adapter](https://docs.pydantic.dev/latest/concepts/type_adapter/)。

整批验证抛出异常，没有返回完整列表。应用可以因此决定整批拒绝；上一种函数则主动保留成功项。**Pydantic 提供验证机制，部分成功还是全部成功由应用选择。** 即便整批验证通过，也不等于数据库写入自动具有事务性。

## 11. 自测

先不运行，写下每题会在哪里失败、或为什么通过：

1. `PlanV1(title="", pages=-1)` 与 `ReadingPlan(title="", pages=-1)` 有什么差别？
2. `note: str | None` 为什么不能省略？怎样改成可以省略？
3. `Field(default_factory=uuid4)` 为什么不能在 `uuid4` 后加括号？
4. `model_dump()` 的返回值为何不保证可以直接 `json.dumps()`？
5. `Field(repr=False)` 的字段是否还会被导出？
6. before 把 `"很多页"` 变成 `"很多"` 后，after 会执行吗？
7. 已读页数等于总页数能否通过？总页数本身不是数字时，是否先比较大小？
8. `.env` 是 40，环境变量是 45，构造参数是 0，最终结果是什么？
9. 已验证模型上的 `tags.append(123)` 是否必然再次触发类型检查？
10. 一批 4 行中有 2 行失败，其中一行包含 3 个字段错误，“失败数量”该怎样报告？

<details markdown="1">
<summary>完成预测后展开答案</summary>

1. `PlanV1` 只有类型要求，因此通过；`ReadingPlan` 有标题长度和页数范围约束，会报告字段错误。
2. 联合类型允许值为 None，没有提供默认值。改为 `note: str | None = None`。
3. 传函数，让模型每次需要时调用；加括号会先执行函数，把结果当工厂。
4. 普通 Python 导出可能包含日期、UUID 等对象。按接收方需求选择 JSON 模式字典或 JSON 文本。
5. 默认仍会。隐藏打印与排除导出分别由不同参数控制。
6. 不会。整数解析失败，后续 after 成功路径不执行。
7. 相等允许，剩余为零；总页数字段先失败时，不会继续执行 after 模型关系检查。
8. 优先选择显式参数 0，然后范围验证失败，不会自动退回低优先级来源。
9. 不会必然检查；容器原地修改并不是对模型属性重新赋值。
10. 分开报告失败行数与错误条目数，避免把“需要修复几行”和“总共有几个问题”混为一谈。

</details>

## 12. 练习：课程学习计划

把场景改成课程学习计划，声明 `course_name`、`total_lessons`、`completed_lessons` 和可省略的备注。先只写类型与默认值，完成一次成功输入和一次类型失败；然后加范围；最后才加“已学课时不能超过总课时”。

完成后继续加入外部别名 `courseName`，生成 JSON 和 Schema，导入三条记录，再用环境变量配置每周学习天数。每增加一项功能，都保留一个有效输入和一个只违反该项规则的输入，这样失败原因不会相互遮挡。

<details markdown="1">
<summary>检查自己的实现是否覆盖了要求</summary>

课程名需要处理纯空白，不能仅凭最小长度判断；总课时用正数约束；已学课时默认 0 且非负；跨字段上限用 after 模型验证器；备注用有默认值的联合类型。别名的输入接受方式与输出 `by_alias` 分开验证。

JSON 往返后比较字段是否一致；Schema 中检查哪些字段必填。批量输入至少放入一条类型错误和一条关系错误，检查错误位置的差异。每周天数限制为 1–7，在启动时验证配置；同时测试 `.env` 与显式参数的覆盖关系。

</details>

相关笔记：[asyncio：从一次等待到有序管理并发任务]({% post_url /dev/python/2026-09-09-asyncio-beginner-notes %})。

## 参考资料

- [Real Python — Pydantic: Simplifying Data Validation in Python](https://realpython.com/python-pydantic/)
- [Pydantic 官方文档](https://docs.pydantic.dev/latest/concepts/models/)
- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
