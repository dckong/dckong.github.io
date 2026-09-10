---
title: "Pydantic：简化 Python 中的数据验证（Real Python 译文）"
author: gpt6_astra
date: 2026-09-09 08:00:00 +0800
last_modified_at: 2026-09-10
categories: [Dev, Python]
tags: [python, pydantic, translation, realpython]
description: "Real Python 教程《Pydantic: Simplifying Data Validation in Python》的中文翻译，涵盖 BaseModel 数据模型、Field 字段约束、自定义验证器、@validate_call 函数参数校验与 pydantic-settings 配置管理。"
toc: true
---

> **译文说明**：本文是 Real Python 教程 [Pydantic: Simplifying Data Validation in Python](https://realpython.com/python-pydantic/)（作者 Harrison Hoffman）的中文翻译，仅供个人学习使用，版权归原作者与 Real Python 所有。原文中的广告、推荐课程、测验推广与订阅引导已删除；代码块按原文示例重新排版，并修正了默认 UUID、字段别名和日期等影响示例结果的问题。
>
> **运行说明**：示例已在 Python 3.13.13、Pydantic 2.13.2、pydantic-settings 2.12.0 与 email-validator 2.3.0 下核对。输出中的随机 UUID、日期和错误提示措辞可能随运行时间或版本变化；较长输出做了换行排版，部分空行与堆栈信息已省略。后文逐步更新同名模型时，请保存文件并重启 Python REPL，再导入新版模型。

Pydantic 是一个功能强大的 Python 数据验证与配置管理库，其设计目标是提升代码库的健壮性与可靠性。从简单的任务（例如检查某个变量是否为整数），到更复杂的任务（例如确保多层嵌套字典的键和值都具有正确的数据类型），Pydantic 几乎能用最少的样板代码处理任何数据验证场景。

**在本教程中，你将学习如何：**

* 使用 Pydantic 的 `BaseModel` 处理**数据模式（data schema）**
* 为复杂场景编写**自定义验证器（custom validator）**
* 使用 Pydantic 的 `@validate_call` **验证函数参数**
* 使用 `pydantic-settings` 管理设置并**配置应用程序**

在本教程中，你会看到 Pydantic 各项功能的第一手示例，读完之后你将拥有扎实的基础，可以应对自己的验证场景。开始之前，最好对 Python 与[面向对象编程](https://realpython.com/python3-object-oriented-programming/)有中等程度的了解。

## Python 的 Pydantic 库

Python 最吸引人的特点之一，在于它是一门动态类型语言。动态类型意味着变量类型在运行时才确定，而不像静态类型语言那样在编译期就显式声明。动态类型非常适合快速开发和易用性，但在真实应用中，你往往需要更健壮的类型检查与数据验证。Python 的 Pydantic 库正是为此而生。

Pydantic 迅速流行起来，如今已成为 Python 中使用最广泛的数据验证库。在第一节中，你会对 Pydantic 有一个整体认识，并预览这个库的强大功能。你还将学会如何安装 Pydantic，以及本教程需要用到的额外依赖。

### 初识 Pydantic（Getting Familiar With Pydantic）

Pydantic 是一个强大的 Python 库，它借助[类型提示](https://realpython.com/python-type-checking/)帮助你轻松验证并序列化数据模式。这让你的代码更健壮、更可读、更简洁，也更容易调试。Pydantic 还能与许多流行的静态类型工具和 IDE 良好集成，让你在运行代码之前就发现模式问题。

Pydantic 的一些显著特性包括：

* **可定制（Customization）**：能用 Pydantic 验证的数据种类几乎没有上限。从 Python 基础类型到高度嵌套的数据结构，Pydantic 让你几乎可以验证和序列化任何 Python 对象。
* **灵活（Flexibility）**：在验证数据时，你可以控制自己想要的严格或宽松程度。有些情况下，你可能希望把传入的数据强制转换为正确的类型，例如接受本应是浮点数、却以整数形式收到的数据；另一些情况下，你可能希望严格强制所接收的数据类型。Pydantic 两种情况都支持。
* **序列化（Serialization）**：你可以把 Pydantic 对象序列化和反序列化为[字典](https://realpython.com/python-dicts/)和 [JSON](https://realpython.com/python-json/) 字符串。这意味着你可以在 Pydantic 对象与 JSON 之间无缝转换。这一能力催生了自文档化的 API，并让它能与几乎所有支持 JSON 模式的工具集成。
* **性能（Performance）**：得益于用 [Rust](https://www.rust-lang.org/) 编写的核心验证逻辑，Pydantic 异常快。这种性能优势带来迅速而可靠的数据处理体验，尤其适合 REST API 这类需要扩展到大量请求的高吞吐应用。
* **生态与行业采用（Ecosystem and Industry Adoption）**：Pydantic 是[许多流行 Python 库](https://docs.pydantic.dev/2.0/why/#ecosystem)的依赖项，例如 [FastAPI](https://realpython.com/fastapi-python-web-apis/)、[LangChain](https://realpython.com/build-llm-rag-chatbot-with-langchain/) 和 [Polars](https://realpython.com/polars-python/)。大多数[大型科技公司](https://docs.pydantic.dev/2.0/why/#using-pydantic)以及众多其他行业也在使用它。这印证了 Pydantic 的社区支持度、可靠性与生命力。

以上只是让 Pydantic 成为极具吸引力的数据验证库的部分关键特性，在本教程中你会看到它们的实际运用。接下来，你会了解如何安装 Pydantic 及其各种依赖。

### 安装 Pydantic（Installing Pydantic）

Pydantic 已发布在 [PyPI](https://pypi.org/) 上，你可以用 [pip](https://realpython.com/what-is-pip/) 安装它。打开终端或命令提示符，创建一个新的虚拟环境，然后运行以下命令安装 Pydantic：

```shell
(venv) $ python -m pip install pydantic
```

这条命令会把 PyPI 上最新版本的 Pydantic 安装到你的机器上。要确认安装成功，启动一个 [Python REPL](https://realpython.com/python-repl/) 并导入 Pydantic：

```python
>>> import pydantic
```

如果导入没有报错，说明你已经成功安装 Pydantic，系统中已经有了 Pydantic 的核心部分。

### 添加可选依赖（Adding Optional Dependencies）

你也可以为 Pydantic 安装可选依赖。例如，本教程会涉及邮箱验证，你可以在安装时一并加入这些依赖：

```shell
(venv) $ python -m pip install "pydantic[email]"
```

Pydantic 还有一个单独的包用于[配置管理](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)，本教程也会讲到。要安装它，请运行以下命令：

```shell
(venv) $ python -m pip install pydantic-settings
```

至此，你已经装好了本教程需要的全部依赖，可以开始探索 Pydantic 了。首先你会学习模型（model）——Pydantic 定义数据模式的主要方式。

## 使用模型（Using Models）

Pydantic 定义数据模式的主要方式是通过[模型](https://docs.pydantic.dev/latest/concepts/models/)。一个 Pydantic 模型是一个对象，类似于 Python 的[数据类](https://realpython.com/python-data-classes/)（dataclass），它用带[注解](https://realpython.com/python-type-checking/#annotations)的字段来定义并保存关于某个实体的数据。与数据类不同的是，Pydantic 的重心在于自动的数据解析、验证与序列化。

理解这一点的最好方式就是自己创建模型，接下来你就来做这件事。

### 使用 Pydantic BaseModel（Working With Pydantic BaseModels）

假设你正在构建一个供人力资源部门管理员工信息的应用，你需要一种方式来确认新员工信息格式正确。例如，每位员工都应有 ID、姓名、邮箱、出生日期、薪资、部门和福利选择。这正是 Pydantic 模型的完美用例！

为了定义员工模型，你创建一个[类](https://realpython.com/python-classes/)，让它[继承](https://realpython.com/inheritance-composition-python/) Pydantic 的 `BaseModel`：

```python
# pydantic_models.py

from datetime import date
from uuid import UUID, uuid4
from enum import Enum

from pydantic import BaseModel, EmailStr, Field


class Department(Enum):
    HR = "HR"
    SALES = "SALES"
    IT = "IT"
    ENGINEERING = "ENGINEERING"


class Employee(BaseModel):
    employee_id: UUID = Field(default_factory=uuid4)
    name: str
    email: EmailStr
    date_of_birth: date
    salary: float
    department: Department
    elected_benefits: bool
```

首先，你导入定义员工模型所需的依赖。接着创建一个[枚举](https://realpython.com/python-enum/)来表示公司中的各个部门，你将用它来注解员工模型中的 `department` 字段。

然后定义 Pydantic 模型 `Employee`，它继承自 `BaseModel`，并通过注解声明员工字段的名称与期望类型。下面逐一说明你在 `Employee` 中定义的每个字段，以及实例化 `Employee` 对象时 Pydantic 如何验证它：

* **`employee_id`**：这是你要保存信息的员工的 [UUID](https://docs.python.org/3/library/uuid.html)。通过 `UUID` 注解，Pydantic 会在创建模型时验证传入的值。`Field(default_factory=uuid4)` 会在每次需要默认值时调用 `uuid4()`，为该实例生成随机 UUID。原文的 `uuid4()` 写法只在定义类时调用一次，会让多个实例复用同一个默认 ID，因此这里使用了默认值工厂。
* **`name`**：员工姓名，Pydantic 期望它是字符串。
* **`email`**：Pydantic 会在底层使用 Python 的 [`email-validator`](https://pypi.org/project/email-validator/) 库来保证每个员工的 `email` 合法。
* **`date_of_birth`**：每位员工的出生日期都必须是合法日期，由 Python [`datetime`](https://realpython.com/python-datetime/) 模块的 `date` 注解。如果你向 `date_of_birth` 传入字符串，Pydantic 会尝试解析并把它转换成 `date` 对象。
* **`salary`**：员工薪资，期望为浮点数。
* **`department`**：每位员工的部门必须是 `HR`、`SALES`、`IT` 或 `ENGINEERING` 之一，也就是你在 `Department` 枚举中定义的值。
* **`elected_benefits`**：该字段保存员工是否选择了福利，Pydantic 期望它是布尔值。

创建 `Employee` 对象最简单的方式，就是像创建其他 Python 对象那样实例化它。为此，打开一个 Python [REPL](https://realpython.com/python-repl/) 并运行以下代码：

```python
>>> from pydantic_models import Employee
>>> Employee(
...     name="Chris DeTuma",
...     email="cdetuma@example.com",
...     date_of_birth="1998-04-02",
...     salary=123_000.00,
...     department="IT",
...     elected_benefits=True,
... )
Employee(employee_id=UUID('73636d47-373b-40cd-a005-4819a84d9ea7'), name='Chris DeTuma', email='cdetuma@example.com', date_of_birth=datetime.date(1998, 4, 2), salary=123000.0, department=<Department.IT: 'IT'>, elected_benefits=True)
```

在这个代码块中，你导入 `Employee` 并用全部必需的员工字段创建了一个对象。Pydantic 成功验证并强制转换了你传入的字段，创建出一个合法的 `Employee` 对象。注意 Pydantic 如何自动把你的日期字符串转换成 `date` 对象，并把 `"IT"` 字符串转换成对应的 `Department` 枚举。

接下来看看当你试图向 `Employee` 实例传入非法数据时，Pydantic 会如何响应：

```python
>>> Employee(
...     employee_id="123",
...     name=False,
...     email="cdetumaexamplecom",
...     date_of_birth="1939804-02",
...     salary="high paying",
...     department="PRODUCT",
...     elected_benefits=300,
... )
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 7 validation errors for Employee
employee_id
  Input should be a valid UUID, invalid length: expected length 32 for simple format, found 3 [type=uuid_parsing, input_value='123', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/uuid_parsing
name
  Input should be a valid string [type=string_type, input_value=False, input_type=bool]
    For further information visit https://errors.pydantic.dev/2.13/v/string_type
email
  value is not a valid email address: An email address must have an @-sign. [type=value_error, input_value='cdetumaexamplecom', input_type=str]
date_of_birth
  Input should be a valid date or datetime, invalid date separator, expected `-` [type=date_from_datetime_parsing, input_value='1939804-02', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/date_from_datetime_parsing
salary
  Input should be a valid number, unable to parse string as a number [type=float_parsing, input_value='high paying', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/float_parsing
department
  Input should be 'HR', 'SALES', 'IT' or 'ENGINEERING' [type=enum, input_value='PRODUCT', input_type=str]
elected_benefits
  Input should be a valid boolean, unable to interpret input [type=bool_parsing, input_value=300, input_type=int]
    For further information visit https://errors.pydantic.dev/2.13/v/bool_parsing
```

在这个例子中，你创建了一个字段数据非法的 `Employee` 对象。Pydantic 为每个字段给出了详细的错误信息，告诉你期望什么、收到了什么，以及可以去哪里进一步了解该错误。

这种细致的验证很强大，因为它阻止你把非法数据存进 `Employee`。它也让你有信心：那些没有报错就实例化出来的 `Employee` 对象确实包含你期望的数据，你可以在后续代码或其他应用中信任这些数据。

Pydantic 的 `BaseModel` 自带一整套方法，让你可以方便地从其他对象（例如字典和 [JSON](https://realpython.com/python-json/)）创建模型。例如，如果你想把字典实例化为 `Employee` 对象，可以使用 `.model_validate()` 这个[类方法](https://realpython.com/instance-class-and-static-methods-demystified/)：

```python
>>> new_employee_dict = {
...     "name": "Chris DeTuma",
...     "email": "cdetuma@example.com",
...     "date_of_birth": "1998-04-02",
...     "salary": 123_000.00,
...     "department": "IT",
...     "elected_benefits": True,
... }
>>> Employee.model_validate(new_employee_dict)
Employee(employee_id=UUID('627ecf40-56dd-48b7-94d8-c30184396ec9'), name='Chris DeTuma', email='cdetuma@example.com', date_of_birth=datetime.date(1998, 4, 2), salary=123000.0, department=<Department.IT: 'IT'>, elected_benefits=True)
```

这里你创建了 `new_employee_dict`，一个包含员工字段的字典，并把它传给 `.model_validate()` 来创建 `Employee` 实例。在底层，Pydantic 会验证每一个字典条目，确保它符合你期望的数据。如果其中有任何数据非法，Pydantic 会像前面那样抛出错误。如果字典缺少某些字段，你同样会收到提示。

对 JSON 对象你也可以用 `.model_validate_json()` 做同样的事：

```python
>>> new_employee_json = """
...  {
...      "employee_id": "d2e7b773-926b-49df-939a-5e98cbb9c9eb",
...      "name": "Eric Slogrenta",
...      "email": "eslogrenta@example.com",
...      "date_of_birth": "1990-01-02",
...      "salary": 125000.0,
...      "department": "HR",
...      "elected_benefits": false
...  }
...  """
>>> new_employee = Employee.model_validate_json(new_employee_json)
>>> new_employee
Employee(employee_id=UUID('d2e7b773-926b-49df-939a-5e98cbb9c9eb'), name='Eric Slogrenta', email='eslogrenta@example.com', date_of_birth=datetime.date(1990, 1, 2), salary=125000.0, department=<Department.HR: 'HR'>, elected_benefits=False)
```

在这个例子中，`new_employee_json` 是一个保存员工字段的合法 JSON 字符串，你用 `.model_validate_json()` 验证并基于 `new_employee_json` 创建了 `Employee` 对象。这看似不起眼，但能够从 JSON 创建并验证 Pydantic 模型其实非常强大，因为 JSON 是网络上最流行的数据传输方式之一。这也是 [FastAPI](https://realpython.com/get-started-with-fastapi/) 依赖 Pydantic 来构建 REST API 的原因之一。

你还可以把 Pydantic 模型序列化为字典和 JSON：

```python
>>> new_employee.model_dump()
{'employee_id': UUID('d2e7b773-926b-49df-939a-5e98cbb9c9eb'), 'name': 'Eric Slogrenta', 'email': 'eslogrenta@example.com', 'date_of_birth': datetime.date(1990, 1, 2), 'salary': 125000.0, 'department': <Department.HR: 'HR'>, 'elected_benefits': False}
>>> new_employee.model_dump_json()
'{"employee_id":"d2e7b773-926b-49df-939a-5e98cbb9c9eb","name":"Eric Slogrenta","email":"eslogrenta@example.com","date_of_birth":"1990-01-02","salary":125000.0,"department":"HR","elected_benefits":false}'
```

这里你用 `.model_dump()` 和 `.model_dump_json()` 分别把 `new_employee` 模型转换成字典和 JSON 字符串。注意 `.model_dump_json()` 返回的 JSON 对象里，`date_of_birth` 和 `department` 都是以字符串形式存储的。

尽管 Pydantic 已经验证了这些字段并把你的模型转换成了 JSON，但下游使用这段 JSON 的人并不知道 `date_of_birth` 必须是合法的 `date`，也不知道 `department` 必须是 `Department` 枚举中的一个类别。为了解决这个问题，你可以从 `Employee` 模型生成一个 [JSON 模式](https://json-schema.org/understanding-json-schema/about)（JSON schema）。

JSON 模式告诉你一个 JSON 对象中期望哪些字段、可以表示哪些值。你可以把它理解为 `Employee` 类定义的 JSON 版本。下面是生成 `Employee` JSON 模式的方法：

```python
>>> Employee.model_json_schema()
{
    '$defs': {
        'Department': {
            'enum': ['HR', 'SALES', 'IT', 'ENGINEERING'],
            'title': 'Department',
            'type': 'string'
        }
    },
    'properties': {
        'employee_id': {
            'format': 'uuid',
            'title': 'Employee Id',
            'type': 'string'
        },
        'name': {'title': 'Name', 'type': 'string'},
        'email': {
            'format': 'email',
            'title': 'Email',
            'type': 'string'
        },
        'date_of_birth': {
            'format': 'date',
            'title': 'Date Of Birth',
            'type': 'string'
        },
        'salary': {'title': 'Salary', 'type': 'number'},
        'department': {'$ref': '#/$defs/Department'},
        'elected_benefits': {'title': 'Elected Benefits', 'type': 'boolean'}
    },
    'required': [
        'name',
        'email',
        'date_of_birth',
        'salary',
        'department',
        'elected_benefits'
    ],
    'title': 'Employee',
    'type': 'object'
}
```

调用 `.model_json_schema()` 时，你会得到一个表示模型 JSON 模式的字典。你看到的第一个条目展示了 `department` 可以取哪些值。你还能看到关于字段应该如何格式化的信息。例如，根据这个 JSON 模式，`employee_id` 期望是 UUID，`date_of_birth` 期望是日期。由于 `employee_id` 的默认值由工厂逐次生成，模式中不会包含一个固定 UUID 的 `default` 值。

你可以用 `json.dumps()` 把 JSON 模式转换成 JSON 字符串，这样几乎[任何编程语言](https://json-schema.org/implementations)都能验证由 `Employee` 模型生成的 JSON 对象。换句话说，Pydantic 不仅能验证传入数据并把它序列化为 JSON，它还通过 JSON 模式为其他编程语言提供了验证你模型数据所需的信息。

到这里，你已经了解了如何使用 Pydantic 的 `BaseModel` 来验证和序列化数据。接下来，你将学习如何使用字段进一步定制验证。

### 使用 Field 定制字段与添加元数据（Using Fields for Customization and Metadata）

到目前为止，你的 `Employee` 模型会验证每个字段的数据类型，并保证 `email`、`date_of_birth`、`department` 等部分字段具有合法格式。不过，假设你还想保证 `salary` 是正数、`name` 不是空字符串、`email` 包含公司的域名。你可以使用 Pydantic 的 [`Field()`](https://docs.pydantic.dev/latest/concepts/fields/) 来实现。

`Field()` 允许你定制模型的字段并为其添加元数据。要了解它的工作方式，请看这个例子：

```python
# pydantic_models.py

from datetime import date
from uuid import UUID, uuid4
from enum import Enum

from pydantic import BaseModel, EmailStr, Field


class Department(Enum):
    HR = "HR"
    SALES = "SALES"
    IT = "IT"
    ENGINEERING = "ENGINEERING"


class Employee(BaseModel):
    employee_id: UUID = Field(default_factory=uuid4, frozen=True)
    name: str = Field(min_length=1, frozen=True)
    email: EmailStr = Field(pattern=r".+@example\.com$")
    date_of_birth: date = Field(alias="birth_date", repr=False, frozen=True)
    salary: float = Field(alias="compensation", gt=0, repr=False)
    department: Department
    elected_benefits: bool
```

这里你用 `Field()` 为部分 `Employee` 字段配置了默认值工厂、验证约束和元数据。下面逐一说明你用来为字段添加额外验证与元数据的 `Field` 参数：

* **`default_factory`**：用它定义一个生成默认值的可调用对象。在上面的例子中，你把 `default_factory` 设为 `uuid4`。这样在需要时就会调用 `uuid4()` 为 `employee_id` 生成一个随机 UUID。你也可以使用 [lambda](https://realpython.com/python-lambda/) 函数来获得更多灵活性。
* **`frozen`**：这是一个布尔参数，设置后可以让字段不可变。也就是说，当 `frozen` 为 `True` 时，对应字段在模型实例化之后就不能再修改。在这个例子中，`employee_id`、`name` 和 `date_of_birth` 都通过 `frozen` 参数变成了不可变字段。
* **`min_length`**：你可以用 `min_length` 和 `max_length` 控制字符串字段的长度。在上面的例子中，你保证 `name` 至少有一个字符。
* **`pattern`**：对于字符串字段，你可以把 `pattern` 设为一个[正则表达式](https://realpython.com/regex-python/)，用来匹配你期望该字段具有的模式。例如，对上例中 `email` 使用那个正则表达式时，Pydantic 会保证每个邮箱都以 `@example.com` 结尾。
* **`alias`**：当你想给字段指定别名时可以使用这个参数。例如，你可以允许把 `date_of_birth` 叫做 `birth_date`，或把 `salary` 叫做 `compensation`。在这个模型的默认配置下，输入时应使用别名；序列化时可用 `model_dump(by_alias=True)` 输出别名。
* **`gt`**：这个参数是 “greater than”（大于）的缩写，用于数值字段来设置下限。在这个例子中，设置 `gt=0` 会在创建或显式验证模型时要求 `salary` 为正数。Pydantic 还有其他[数值约束](https://docs.pydantic.dev/latest/concepts/fields/#numeric-constraints)，例如 `lt`，即 “less than”（小于）的缩写。
* **`repr`**：这个布尔参数决定字段是否显示在模型的字段表示中。在这个例子中，打印 `Employee` 实例时你看不到 `date_of_birth` 和 `salary`。

要查看这些额外验证的实际效果，注意当你试图用错误数据创建 `Employee` 模型时会发生什么：

```python
>>> from pydantic_models import Employee
>>> incorrect_employee_data = {
...     "name": "",
...     "email": "cdetuma@fakedomain.com",
...     "birth_date": "1998-04-02",
...     "compensation": -10,
...     "department": "IT",
...     "elected_benefits": True,
... }
>>> Employee.model_validate(incorrect_employee_data)
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 3 validation errors for Employee
name
  String should have at least 1 character [type=string_too_short, input_value='', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_too_short
email
  String should match pattern '.+@example\.com$' [type=string_pattern_mismatch, input_value='cdetuma@fakedomain.com', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_pattern_mismatch
compensation
  Input should be greater than 0 [type=greater_than, input_value=-10, input_type=int]
    For further information visit https://errors.pydantic.dev/2.13/v/greater_than
```

这里你导入了更新后的 `Employee` 模型，并尝试验证一个包含错误数据的字典。作为回应，Pydantic 给出了三个验证错误，说明 `name` 至少需要一个字符、`email` 应匹配公司域名、`salary` 应大于零。

现在再来看看当验证正确的 `Employee` 数据时，你能获得哪些额外功能：

```python
>>> employee_data = {
...     "name": "Clyde Harwell",
...     "email": "charwell@example.com",
...     "birth_date": "2000-06-12",
...     "compensation": 100_000,
...     "department": "ENGINEERING",
...     "elected_benefits": True,
... }
>>> employee = Employee.model_validate(employee_data)
>>> employee
Employee(employee_id=UUID('614c6f75-8528-4272-9cfc-365ddfafebd9'), name='Clyde Harwell', email='charwell@example.com', department=<Department.ENGINEERING: 'ENGINEERING'>, elected_benefits=True)
>>> employee.salary
100000.0
>>> employee.date_of_birth
datetime.date(2000, 6, 12)
```

在这个代码块中，你创建了一个字典并通过 `.model_validate()` 创建了 `Employee` 模型。在 `employee_data` 中，注意你用 `birth_date` 代替了 `date_of_birth`，用 `compensation` 代替了 `salary`。Pydantic 能识别这些别名，并在内部把它们的值赋给正确的字段名。

由于你设置了 `repr=False`，可以看到 `salary` 和 `date_of_birth` 并没有显示在 `Employee` 的表示中。你必须显式地把它们当作属性访问才能看到值。最后，注意当你试图修改冻结字段时会发生什么：

```python
>>> employee.department = "HR"
>>> employee.name = "Andrew TuGrendele"
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 1 validation error for Employee
name
  Field is frozen [type=frozen_field, input_value='Andrew TuGrendele', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/frozen_field
```

这里你先把 `department` 从 `Department.ENGINEERING` 改成字符串 `"HR"`。赋值能够成功，因为该字段没有冻结，而且 Pydantic 默认不会重新验证属性赋值，因此这里也不会自动把字符串转换成枚举。若希望修改属性时仍进行验证，可以在模型配置中设置 `validate_assignment=True`。当你试图修改 `name` 时，Pydantic 则会报错，因为 `name` 是冻结字段。

现在你已经扎实掌握了 Pydantic 的 `BaseModel` 与 `Field()`。仅凭它们，你就能在数据模式上定义许多不同的验证规则和元数据，但有时这还不够。接下来，你会用 Pydantic 验证器把字段验证推进得更远。

## 使用验证器（Working With Validators）

到目前为止，你已经用 Pydantic 的 `BaseModel` 配合预定义类型来验证模型字段，并用 `Field` 进一步定制验证。仅靠 `BaseModel` 和 `Field` 你已经能走得很远，但对于需要自定义逻辑的更复杂验证场景，你需要使用 Pydantic 的[验证器](https://docs.pydantic.dev/latest/concepts/validators/)（validator）。

有了验证器，你几乎可以执行任何能用函数表达的验证逻辑。接下来你会看到具体做法。

### 验证模型与字段（Validating Models and Fields）

继续沿用员工的例子，假设你的公司有政策：只雇佣年满十八岁的员工。每次创建新的 `Employee` 对象时，你都需要确保该员工至少十八岁。为此，你可以添加一个 `age` 字段并用 `Field()` 强制要求员工至少十八岁。但既然你已经存了员工的出生日期，这样做就显得多余了。

更好的方案是使用 Pydantic 的[字段验证器](https://docs.pydantic.dev/latest/concepts/validators/#field-validators)（field validator）。字段验证器让你可以通过向模型添加类方法，把自定义验证逻辑应用到 `BaseModel` 的字段上。要强制所有员工至少十八岁，你可以给 `Employee` 模型添加下面这个字段验证器：

```python
# pydantic_models.py

from datetime import date
from uuid import UUID, uuid4
from enum import Enum

from pydantic import BaseModel, EmailStr, Field, field_validator


class Department(Enum):
    HR = "HR"
    SALES = "SALES"
    IT = "IT"
    ENGINEERING = "ENGINEERING"


class Employee(BaseModel):
    employee_id: UUID = Field(default_factory=uuid4, frozen=True)
    name: str = Field(min_length=1, frozen=True)
    email: EmailStr = Field(pattern=r".+@example\.com$")
    date_of_birth: date = Field(alias="birth_date", repr=False, frozen=True)
    salary: float = Field(alias="compensation", gt=0, repr=False)
    department: Department
    elected_benefits: bool

    @field_validator("date_of_birth")
    @classmethod
    def check_valid_age(cls, date_of_birth: date) -> date:
        today = date.today()
        eighteen_years_ago = date(today.year - 18, today.month, today.day)

        if date_of_birth > eighteen_years_ago:
            raise ValueError("Employees must be at least 18 years old.")

        return date_of_birth
```

在这个代码块中，你导入了 `field_validator`，并用它装饰 `Employee` 中名为 `.check_valid_age()` 的类方法。这里使用类方法作为字段验证器，并显式加上 `@classmethod`，让方法签名更清晰。在 `.check_valid_age()` 中，你计算出十八年前的今天。如果员工的 `date_of_birth` 晚于该日期，就抛出错误。

要看这个验证器如何工作，请看这个例子：

```python
>>> from pydantic_models import Employee
>>> from datetime import date, timedelta
>>> young_employee_data = {
...     "name": "Jake Bar",
...     "email": "jbar@example.com",
...     "birth_date": date.today() - timedelta(days=365 * 17),
...     "compensation": 90_000,
...     "department": "SALES",
...     "elected_benefits": True,
... }
>>> Employee.model_validate(young_employee_data)
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 1 validation error for Employee
birth_date
  Value error, Employees must be at least 18 years old. [type=value_error, input_value=datetime.date(2009, 9, 14), input_type=date]
    For further information visit https://errors.pydantic.dev/2.13/v/value_error
```

在这个例子中，你指定的 `birth_date` 比当前日期早约十七年；上面的日期输出以 2026-09-10 运行时为例。当你调用 `.model_validate()` 验证 `young_employee_data` 时，会得到错误提示：员工必须至少十八岁。

可以想象，Pydantic 的 `field_validator()` 让你能任意定制字段验证。如果你想比较多个字段之间的关系，或者把模型作为一个整体来验证，使用[模型验证器](https://docs.pydantic.dev/latest/concepts/validators/#model-validators)（model validator）通常更直接。字段验证器也能通过 `ValidationInfo.data` 访问先前已验证的字段，但会受字段定义顺序影响。

举个例子，假设你的公司只在 IT 部门雇佣合同工。因此 IT 员工不符合福利条件，他们的 `elected_benefits` 字段应该是 `False`。你可以用 Pydantic 的 `model_validator()` 来强制这条约束：

```python
# pydantic_models.py

from typing import Self
from datetime import date
from uuid import UUID, uuid4
from enum import Enum

from pydantic import (
    BaseModel,
    EmailStr,
    Field,
    field_validator,
    model_validator,
)


class Department(Enum):
    HR = "HR"
    SALES = "SALES"
    IT = "IT"
    ENGINEERING = "ENGINEERING"


class Employee(BaseModel):
    employee_id: UUID = Field(default_factory=uuid4, frozen=True)
    name: str = Field(min_length=1, frozen=True)
    email: EmailStr = Field(pattern=r".+@example\.com$")
    date_of_birth: date = Field(alias="birth_date", repr=False, frozen=True)
    salary: float = Field(alias="compensation", gt=0, repr=False)
    department: Department
    elected_benefits: bool

    @field_validator("date_of_birth")
    @classmethod
    def check_valid_age(cls, date_of_birth: date) -> date:
        today = date.today()
        eighteen_years_ago = date(today.year - 18, today.month, today.day)

        if date_of_birth > eighteen_years_ago:
            raise ValueError("Employees must be at least 18 years old.")

        return date_of_birth

    @model_validator(mode="after")
    def check_it_benefits(self) -> Self:
        department = self.department
        elected_benefits = self.elected_benefits

        if department == Department.IT and elected_benefits:
            raise ValueError(
                "IT employees are contractors and don't qualify for benefits"
            )

        return self
```

这里你把 Python 的 [`Self` 类型](https://realpython.com/python-type-self/)和 Pydantic 的 `model_validator()` 加入了导入。然后创建一个方法 `.check_it_benefits()`，当员工属于 IT 部门且 `elected_benefits` 字段为 `True` 时抛出错误。当你在 `@model_validator` 中把 `mode` 设为 `after` 时，Pydantic 会等到模型实例化完成之后再运行 `.check_it_benefits()`。

**注意**：你可能已经注意到 `.check_it_benefits()` 用了 Python 的 `Self` 类型做注解。这是因为 `.check_it_benefits()` 返回的是 `Employee` 类实例，而 `Self` 类型是这种情况下的[首选注解](https://realpython.com/python-type-self/#how-to-annotate-a-method-with-the-self-type-in-python)。如果你使用的 Python 版本低于 3.11，就需要从 `typing_extensions` 导入 `Self` 类型。

要看你的新模型验证器如何工作，请看这个例子：

```python
>>> from pydantic_models import Employee
>>> new_employee = {
...     "name": "Alexis Tau",
...     "email": "ataue@example.com",
...     "birth_date": "2001-04-12",
...     "compensation": 100_000,
...     "department": "IT",
...     "elected_benefits": True,
... }
>>> Employee.model_validate(new_employee)
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 1 validation error for Employee
  Value error, IT employees are contractors and don't qualify for benefits [type=value_error, input_value={'name': 'Alexis Tau', ...elected_benefits': True}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.13/v/value_error
```

在这个例子中，你试图创建一个 `department` 为 `IT`、`elected_benefits` 为 `True` 的 `Employee` 模型。调用 `.model_validate()` 时，Pydantic 抛出错误，告诉你 IT 员工是合同工，不符合福利条件。

有了模型验证器和字段验证器，你几乎可以实现任何能想到的自定义验证。现在你应该已经具备扎实的基础，可以为自己的用例创建 Pydantic 模型了。接下来换个方向，看看如何用 Pydantic 验证任意函数，而不仅仅是 `BaseModel` 的字段。

### 使用验证装饰器验证函数（Using Validation Decorators to Validate Functions）

`BaseModel` 是 Pydantic 用于验证数据模式的核心类，但你也可以使用 [`@validate_call`](https://docs.pydantic.dev/latest/concepts/validation_decorator/) 装饰器来验证函数参数。这让你无需手动实现验证逻辑，就能写出带有丰富类型错误提示的健壮函数。

要了解它的工作方式，假设你在写一个函数：客户完成购买后向其发送发票。该函数接收客户姓名、邮箱、购买的商品以及账单总额，然后构造并发送邮件。你需要验证所有这些输入，因为一旦出错，可能导致邮件发不出去、格式错误，或者给客户开错账单。

为此，你编写如下函数：

```python
# validate_functions.py

import time
from typing import Annotated

from pydantic import PositiveFloat, Field, EmailStr, validate_call


@validate_call
def send_invoice(
    client_name: Annotated[str, Field(min_length=1)],
    client_email: EmailStr,
    items_purchased: list[str],
    amount_owed: PositiveFloat,
) -> str:
    email_str = f"""
    Dear {client_name},
    Thank you for choosing xyz inc!
    You owe ${amount_owed:,.2f} for the following items:
    {items_purchased}
    """
    print(f"Sending email to {client_email}...")
    time.sleep(2)
    return email_str
```

首先，你导入编写和注解 `send_invoice()` 所需的依赖。然后创建一个用 `@validate_call` 装饰的 `send_invoice()`。在执行 `send_invoice()` 之前，`@validate_call` 会确保每个输入都符合你的注解。在这个例子中，`@validate_call` 会检查 `client_name` 是否至少有一个字符、`client_email` 格式是否正确、`items_purchased` 是否为字符串列表、`amount_owed` 是否为正浮点数。

如果某个输入不符合你的注解，Pydantic 会抛出与你之前在 `BaseModel` 中看到的类似错误。如果所有输入都合法，`send_invoice()` 会创建一个字符串，并用 `time.sleep(2)` 模拟把它发送给客户。

**注意**：你可能已经注意到 `client_name` 用了 Python 的 `Annotated` 类型做注解。一般来说，当你想为函数参数提供元数据时可以使用 `Annotated`。当需要验证带有 `Field` 指定元数据的函数参数时，Pydantic [推荐](https://docs.pydantic.dev/latest/concepts/validation_decorator/#using-field-to-describe-function-arguments)使用 `Annotated`。

不过，如果你想用 `default_factory` 为函数参数指定默认值，就应该把参数直接赋值为一个 `Field` 实例。[Pydantic 文档](https://docs.pydantic.dev/latest/concepts/validation_decorator/#using-field-to-describe-function-arguments)中有这样的例子。

要看 `@validate_call` 和 `send_invoice()` 的实际效果，打开一个新的 Python REPL 并运行以下代码：

```python
>>> from validate_functions import send_invoice
>>> send_invoice(
...     client_name="",
...     client_email="ajolawsonfakedomain.com",
...     items_purchased=["pie", "cookie", 17],
...     amount_owed=0,
... )
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 4 validation errors for send_invoice
client_name
  String should have at least 1 character [type=string_too_short, input_value='', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_too_short
client_email
  value is not a valid email address: An email address must have an @-sign. [type=value_error, input_value='ajolawsonfakedomain.com', input_type=str]
items_purchased.2
  Input should be a valid string [type=string_type, input_value=17, input_type=int]
    For further information visit https://errors.pydantic.dev/2.13/v/string_type
amount_owed
  Input should be greater than 0 [type=greater_than, input_value=0, input_type=int]
    For further information visit https://errors.pydantic.dev/2.13/v/greater_than
```

在这个例子中，你导入 `send_invoice()` 并传入非法的函数参数。Pydantic 的 `@validate_call` 识别出这一点并抛出错误，告诉你 `client_name` 至少需要一个字符、`client_email` 非法、`items_purchased` 应包含字符串、`amount_owed` 应大于零。

当你传入合法输入时，`send_invoice()` 会按预期运行：

```python
>>> email_str = send_invoice(
...     client_name="Andrew Jolawson",
...     client_email="ajolawson@fakedomain.com",
...     items_purchased=["pie", "cookie", "cake"],
...     amount_owed=20,
... )
Sending email to ajolawson@fakedomain.com...
>>> print(email_str)
    Dear Andrew Jolawson,
    Thank you for choosing xyz inc!
    You owe $20.00 for the following items:
    ['pie', 'cookie', 'cake']
```

虽然 `@validate_call` 不如 `BaseModel` 灵活，但你仍然可以用它给函数参数施加强大的验证。这能为你节省大量时间，让你不必编写样板式的类型检查与验证逻辑。如果你以前做过这类事情，就会知道为每个函数参数编写 [`assert` 语句](https://realpython.com/python-assert-statement/)有多么繁琐。在许多用例中，`@validate_call` 会替你完成这件事。

在本教程的最后一节，你将学习如何使用 Pydantic 进行配置管理与应用程序配置。

## 管理配置（Managing Settings）

配置 Python 应用最流行的方式之一就是使用环境变量。环境变量存在于操作系统中、位于你的 Python 代码之外，但可以被你的代码或其他程序读取。适合存为环境变量的数据包括密钥、数据库凭据、API 凭据、服务器地址和访问令牌。

环境变量在开发和生产之间经常变化，而且很多包含敏感信息。因此，你需要一种健壮的方式来解析、验证并集成环境变量。这正是 `pydantic-settings` 的完美用例，也是本节要探讨的内容。

### 使用 BaseSettings 配置应用（Configuring Applications With BaseSettings）

`pydantic-settings` 是在 Python 中管理环境变量最强大的方式之一，它已被 [FastAPI](https://fastapi.tiangolo.com/advanced/settings/#pydantic-settings) 等流行库广泛采用和推荐。你可以用 `pydantic-settings` 创建类似 `BaseModel` 的模型，用来解析和验证环境变量。

`pydantic-settings` 中的主类是 `BaseSettings`，它拥有与 `BaseModel` 完全相同的功能。不过，如果你创建的模型继承自 `BaseSettings`，模型初始化器会尝试从未作为关键字参数传入的环境变量中读取字段的值。

要了解它的工作方式，假设你的应用要连接一个数据库和另一个 API 服务。你的数据库凭据和 API 密钥会随时间变化，也常常随部署环境的不同而变化。为此，你可以创建下面这个 `BaseSettings` 模型：

```python
# settings_management.py

from pydantic import HttpUrl, Field
from pydantic_settings import BaseSettings


class AppConfig(BaseSettings):
    database_host: HttpUrl
    database_user: str = Field(min_length=5)
    database_password: str = Field(min_length=10)
    api_key: str = Field(min_length=20)
```

在这个脚本中，你导入了创建 `BaseSettings` 模型所需的依赖。注意你是用下划线而不是连字符从 `pydantic_settings` 导入 `BaseSettings`。接着定义模型 `AppConfig`，它继承自 `BaseSettings`，保存数据库和 API 密钥相关字段。在这个例子中，`database_host` 必须是合法的 HTTP URL，其余字段都有最小长度约束。

接下来打开终端，添加以下环境变量。如果你使用 Linux、macOS 或 Windows Bash，可以用 `export` 命令完成：

```shell
(venv) $ export DATABASE_HOST="http://somedatabaseprovider.us-east-2.com"
(venv) $ export DATABASE_USER="username"
(venv) $ export DATABASE_PASSWORD="asdfjl348ghl@9fhsl4"
(venv) $ export API_KEY="ajfsdla48fsdal49fj94jf93-f9dsal"
```

你也可以[在 Windows PowerShell 中设置环境变量](https://realpython.com/python-coding-setup-windows/#setting-and-changing-environment-variables)。然后打开一个新的 Python REPL 并实例化 `AppConfig`：

```python
>>> from settings_management import AppConfig
>>> AppConfig()
AppConfig(database_host=HttpUrl('http://somedatabaseprovider.us-east-2.com/'), database_user='username', database_password='asdfjl348ghl@9fhsl4', api_key='ajfsdla48fsdal49fj94jf93-f9dsal')
```

注意实例化 `AppConfig` 时你没有指定任何字段名。相反，你的 `BaseSettings` 模型从你设置的环境变量中读取字段。还要注意你用全大写导出了环境变量，而 `AppConfig` 依然成功解析并保存了它们。这是因为 `BaseSettings` 在把环境变量匹配到字段名时不区分大小写。

接下来关闭 Python REPL，并创建非法的环境变量：

```shell
(venv) $ export DATABASE_HOST="somedatabaseprovider.us-east-2"
(venv) $ export DATABASE_USER="usee"
(venv) $ export DATABASE_PASSWORD="asdf"
(venv) $ export API_KEY="ajf"
```

现在打开另一个 Python REPL 并重新实例化 `AppConfig`：

```python
>>> from settings_management import AppConfig
>>> AppConfig()
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 4 validation errors for AppConfig
database_host
  Input should be a valid URL, relative URL without a base [type=url_parsing, input_value='somedatabaseprovider.us-east-2', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/url_parsing
database_user
  String should have at least 5 characters [type=string_too_short, input_value='usee', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_too_short
database_password
  String should have at least 10 characters [type=string_too_short, input_value='asdf', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_too_short
api_key
  String should have at least 20 characters [type=string_too_short, input_value='ajf', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/string_too_short
```

这一次，当你试图实例化 `AppConfig` 时，`pydantic-settings` 抛出的错误说明 `database_host` 不是合法 URL，其余字段不满足最小长度约束。

这只是一个简化的配置示例，但你可以借助 `BaseSettings` 解析并验证环境变量中几乎所有需要的内容。你能用 `BaseModel` 做的任何验证，都可以用 `BaseSettings` 完成，包括使用模型验证器和字段验证器进行自定义验证。

最后，你将学习如何用 `SettingsConfigDict` 进一步定制 `BaseSettings` 的行为。

### 使用 SettingsConfigDict 定制配置（Customizing Settings With SettingsConfigDict）

在之前的例子里，你已经看到了创建 `BaseSettings` 模型来解析和验证环境变量的最简形式。不过，你可能想进一步定制 `BaseSettings` 模型的行为，这可以用 `SettingsConfigDict` 来实现。

假设你无法手动导出每一个环境变量（这种情况很常见），而是需要从一个 [`.env`](https://pypi.org/project/python-dotenv/) 文件读取它们。你可能希望确保 `BaseSettings` 在解析时区分大小写，并且 `.env` 文件中除了模型里指定的变量之外没有其他环境变量。下面是你用 `SettingsConfigDict` 实现的方式：

```python
# settings_management.py

from pydantic import HttpUrl, Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppConfig(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
        extra="forbid",
    )

    database_host: HttpUrl
    database_user: str = Field(min_length=5)
    database_password: str = Field(min_length=10)
    api_key: str = Field(min_length=20)
```

这个脚本与上一个例子相同，只是这一次你导入了 `SettingsConfigDict` 并在 `AppConfig` 中初始化了它。在 `SettingsConfigDict` 里，你指定环境变量应从 `.env` 文件读取、强制区分大小写，并且禁止 `.env` 文件中出现额外的环境变量。

接下来，在 `settings_management.py` 所在的同一目录下创建一个名为 `.env` 的文件，并从该目录启动 Python。这里的 `env_file=".env"` 相对于当前工作目录解析，而不是自动相对于模块文件。已有的同名系统环境变量会优先于 `.env` 中的值。文件内容如下：

```text
database_host=http://somedatabaseprovider.us-east-2.com/
database_user=username
database_password=asdfjfffffl348ghl@9fhsl4
api_key=ajfsdla48fsdal49fj94jf93-f9dsal
```

现在，你可以打开 Python REPL 并初始化 `AppConfig` 模型：

```python
>>> from settings_management import AppConfig
>>> AppConfig()
AppConfig(database_host=HttpUrl('http://somedatabaseprovider.us-east-2.com/'), database_user='username', database_password='asdfjfffffl348ghl@9fhsl4', api_key='ajfsdla48fsdal49fj94jf93-f9dsal')
```

如你所见，`AppConfig` 成功解析并验证了 `.env` 文件中的环境变量。

最后，向 `.env` 文件中添加一些非法变量：

```text
DATABASE_HOST=http://somedatabaseprovider.us-east-2.com/
database_user=username
database_password=asdfjfffffl348ghl@9fhsl4
api_key=ajfsdla48fsdal49fj94jf93-f9dsal
extra_var=shouldntbehere
```

这里你把 `database_host` 改成了 `DATABASE_HOST`，违反了区分大小写的约束，并且添加了本不该存在的额外环境变量。下面是模型在尝试验证时给出的响应：

```python
>>> from settings_management import AppConfig
>>> AppConfig()
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 3 validation errors for AppConfig
database_host
  Field required [type=missing, input_value={'database_user': 'userna..._var': 'shouldntbehere'}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.13/v/missing
DATABASE_HOST
  Extra inputs are not permitted [type=extra_forbidden, input_value='http://somedatabaseprovider.us-east-2.com/', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/extra_forbidden
extra_var
  Extra inputs are not permitted [type=extra_forbidden, input_value='shouldntbehere', input_type=str]
    For further information visit https://errors.pydantic.dev/2.13/v/extra_forbidden
```

你得到了一份清晰的错误列表，说明 `database_host` 缺失，并且 `.env` 文件中有额外的环境变量。注意由于区分大小写的约束，模型认为 `DATABASE_HOST` 与 `extra_var` 一样是额外变量。

关于 `SettingsConfigDict` 以及更广义的 `BaseSettings`，还有很多可以做的事，但这些例子应该足以让你了解如何用 `pydantic-settings` 为自己的用例管理环境变量。

## 结论（Conclusion）

Pydantic 是一个易用、快速且广受信任的 Python 数据验证库。你已经对 Pydantic 有了全面的了解，现在具备了在自己的项目中使用 Pydantic 所需的知识与资源。

**在本教程中，你学到了**：

* **Pydantic** 是什么，以及它为何被如此广泛地采用
* 如何**安装** Pydantic
* 如何用 **`BaseModel`** 和**验证器**解析、验证并序列化数据模式
* 如何使用 **@validate\_call** 为函数编写自定义验证逻辑
* 如何用 **`pydantic-settings`** 解析和验证环境变量

Pydantic 让你的代码更健壮、更值得信赖，也在一定程度上弥合了 Python 的易用性与静态类型语言内置数据验证之间的差距。对于几乎任何数据解析、验证和序列化用例，Pydantic 都提供了优雅的解决方案。

如果你需要检查整张数据表，而不是一次一条记录，那么可以看看 [Validating Data With Pointblank in Python](https://realpython.com/python-pointblank/)。
