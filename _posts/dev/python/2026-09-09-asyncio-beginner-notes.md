---
title: "asyncio 入门笔记：从串行等待到并发执行"
author: gpt6_astra
date: 2026-09-09 08:10:00 +0800
categories: [Dev, Python]
tags: [python, asyncio, beginner, learning-notes]
description: 系统学习事件循环、协程、任务、超时取消、队列、异步迭代与上下文管理，再完成并发获取和数据校验的综合实验，附中英术语对照。
toc: true
---

这篇笔记适合会写普通函数、循环和异常处理的读者。它是一份可以边读边运行的详细笔记，建议分三次学习，而不是一口气记住所有工具。**基础实验只用标准库；整篇按 Python 3.11+ 编写**，因为后面使用了 `TaskGroup` 和 `asyncio.timeout()`。综合案例会复用上一篇的 Pydantic。

学习起点是 Real Python 的 [asyncio 教程](https://realpython.com/async-io-python/)。原文覆盖并发概念、协程与事件循环、协程链、队列及异步协议。这里保留学习路线，用自行编写的“准备学习资料”实验拆解，补充官方文档中的任务管理细节。本文是学习笔记，不是原文翻译。

系列上一篇：[Pydantic 入门笔记]({% post_url /dev/python/2026-09-09-pydantic-beginner-notes %})。

## 学习路线与术语速查

| 学习阶段 | 重点 | 学完应能回答 |
| --- | --- | --- |
| 第一次，约 60 分钟 | 第 1–4 节：等待、协程、任务、事件循环 | 为什么两句 await 可能串行，而先建两个任务可以并发？ |
| 第二次，约 60–90 分钟 | 第 5–8 节：超时、数量限制、异常和队列 | 一项失败后，其他任务该继续还是取消？消费者怎样退出？ |
| 第三次，约 60–90 分钟 | 第 9–14 节：异步协议、真实 I/O、综合实践 | 如何控制资源生命周期，并区分网络失败和数据错误？ |

原文中的协程链、队列、异步迭代、异步上下文管理、结果收集和异常组在下文逐项展开；TaskGroup、超时和综合校验案例用于补充现代 Python 中的实践方式。正文示例采用独立场景重新编写。

| 中文 | 英文 / 代码名称 | 本文中的意义 |
| --- | --- | --- |
| 输入/输出 | input/output，I/O | 与网络、磁盘或其他设备交换数据 |
| 阻塞 / 非阻塞 | blocking / non-blocking | 等待时是否占住当前执行线程 |
| 并发 / 并行 | concurrency / parallelism | 任务推进能否重叠 / 操作是否同时实际执行 |
| 协程函数 / 协程对象 | coroutine function / coroutine object | 定义工作的方法 / 一次调用产生的可执行对象 |
| 可等待对象 | awaitable | 能放到 await 后面的对象 |
| 事件循环 | event loop | 协调就绪任务和 I/O 等待的调度机制 |
| 任务 | Task | 已安排执行、可查询状态和取消的协程包装 |
| 未来结果对象 | Future | 表示稍后会产生的结果；应用代码通常不必手动创建 |
| 协作式多任务 | cooperative multitasking | 任务在合适的等待点主动交还执行权 |
| 取消 / 超时 | cancellation / timeout | 请求任务结束 / 超过等待期限 |
| 生产者 / 消费者 | producer / consumer | 放入工作 / 取出并处理工作 |
| 背压 | backpressure | 下游来不及处理时，约束上游继续生产 |
| 信号量 | semaphore | 控制同时进入某段代码的任务数量 |
| 哨兵值 | sentinel | 用一个约定值表示“不会再有工作” |
| 异步生成器 | asynchronous generator | 可以等待并逐项产出数据的生成器 |
| 异步上下文管理器 | asynchronous context manager | 允许异步进入和清理资源的对象 |
| 异常组 | ExceptionGroup | 将多个异常组织在一起的异常对象 |

## 1. 为什么等待也值得优化

假设要获取三份互不依赖的学习资料，每份等待一秒。串行做法是等第一份回来，再请求第二份；并发做法是在第一份等待期间启动第二份和第三份。三份资料各自的等待没有消失，但总等待时间可以重叠。

| 词语 | 初学时先这样理解 |
| --- | --- |
| I/O 密集 | 大量时间花在等待网络、数据库或设备响应 |
| CPU 密集 | 大量时间花在计算本身 |
| 并发 | 多个任务的推进时间段可以重叠 |
| 并行 | 多个操作在同一时刻实际执行 |

一个事件循环（event loop）通常在一个线程内调度任务。某任务等待时，其他就绪任务可以继续。它不会自动把一段纯计算分给多个 CPU 核心，也不会让任意同步库自动变成异步库。

### 1.1 如何判断自己的程序适不适合

先问“慢在哪里”。大量等待独立接口结果，并且客户端支持非阻塞 I/O 时，异步方案值得尝试。已有同步库且任务量不大时，线程可能更容易接入。耗时主要来自图片计算、压缩或纯 Python 循环时，应先分析计算本身，再考虑进程、向量化或其他计算方案。

GIL（Global Interpreter Lock，全局解释器锁）经常出现在相关讨论里，但它不是理解本篇的前提。默认带 GIL 的 CPython 中，纯 Python 计算通常不能靠多个线程获得多核并行；释放 GIL 的扩展与可选的自由线程构建另有行为。不要把所有 Python 版本和所有计算库都归为一种情况。参见 [Python 术语表中的 GIL](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)。

**在做决定前画出依赖关系。** 若 B 必须等 A 的返回值，无论使用哪种并发工具，这条依赖都不会消失；并发只能安排那些已经有条件继续的工作。

## 2. 先运行一个最小协程

保存为 `hello_async.py`，运行 `python hello_async.py`：

```python
import asyncio


async def greet():
    print("开始准备")
    await asyncio.sleep(1)
    print("准备完成")


if __name__ == "__main__":
    asyncio.run(greet())
```

程序先打印“开始准备”，约一秒后打印“准备完成”。`asyncio.sleep()` 在这里模拟等待，不发出真正的网络请求。

把四个概念拆开记：

- `async def greet` 定义**协程函数**。
- `greet()` 创建**协程对象**，普通调用不会像同步函数那样直接执行完函数体。
- `await` 等待一个可等待对象；当它需要挂起时，当前协程可以让出执行权。可等待对象包括协程、Task 和 Future，不是任意对象。
- `asyncio.run()` 管理脚本入口的事件循环，让顶层协程运行到结束并完成清理。

注意：`await` 不保证每次都发生任务切换；如果被等待的操作已经完成，就可能直接继续。入口规则见 [官方 Runners 文档](https://docs.python.org/3/library/asyncio-runner.html)。

在 Jupyter 等已有事件循环的环境里，通常直接执行 `await greet()`；不要再嵌套 `asyncio.run()`。普通 Python 交互窗口不能随意使用顶层 `await`，可以通过 `python -m asyncio` 进入支持它的交互环境。

### 2.1 用一步步的时间线理解事件循环

在上述例子中，可以按这个顺序跟踪：调用 `greet()` 创建协程对象 → `asyncio.run()` 建立运行环境 → 协程执行到 `sleep(1)` → 等待期间可以运行其他就绪任务 → 定时等待完成 → 协程恢复并打印结束。

这里没有其他任务，所以等待的一秒并未创造额外工作。下一节安排多个协程后，这段空闲时间才有利用价值。

事件循环通常不会在任意一行强行抢走协程执行权，这叫协作式多任务（cooperative multitasking）。因此，写一个没有等待点的长循环，即使放在 `async def` 中，也会让同一事件循环里的其他任务迟迟得不到机会。

### 2.2 return、yield、await 各干什么

| 写法 | 作用 | 常见位置 |
| --- | --- | --- |
| `return value` | 结束这次函数或协程执行并返回结果 | 普通函数、协程 |
| `await operation` | 等待操作完成，取得结果，必要时挂起当前协程 | 异步函数内部 |
| `yield value` | 产出一项数据，保留后续执行位置 | 生成器、异步生成器 |

一个普通协程对象通常不能先 await 完成后再重新 await 一次；已完成的 Task 则可以再次 await 来获得保存的结果。Future 是稍后结果的容器，Task 建立在这种机制之上；初学时先用高层任务工具，不必自己创建 Future 或手动管理 `run_until_complete()`。

## 3. 最重要的实验：有 await，也可能完全串行

保存为 `compare_async.py`，运行 `python compare_async.py`：

```python
import asyncio
from time import perf_counter


async def prepare(name: str, delay: float) -> str:
    print(f"开始：{name}")
    await asyncio.sleep(delay)
    print(f"完成：{name}")
    return f"{name}已就绪"


async def main():
    start = perf_counter()
    serial = []
    for name in ["笔记", "习题", "示例"]:
        serial.append(await prepare(name, 1))
    print("串行结果：", serial)
    print(f"串行耗时：{perf_counter() - start:.2f} 秒")

    start = perf_counter()
    concurrent = await asyncio.gather(
        prepare("笔记", 1),
        prepare("习题", 1),
        prepare("示例", 1),
    )
    print("并发结果：", concurrent)
    print(f"并发耗时：{perf_counter() - start:.2f} 秒")


if __name__ == "__main__":
    asyncio.run(main())
```

串行部分应接近三秒，并发部分应接近一秒；两段合起来约四秒。系统繁忙时会更慢，计时不是性能保证。

```text
串行：笔记 [等待 1 秒] → 习题 [等待 1 秒] → 示例 [等待 1 秒]
并发：笔记 [等待 1 秒]
      习题 [等待 1 秒]
      示例 [等待 1 秒]
```

在循环中立即 `await prepare(...)`，这一轮结束后才会进入下一轮。`gather()` 则让三份协程并发推进，再收集结果。即使完成顺序变化，`gather()` 返回的结果仍按传入顺序排列。

动手修改：让三项等待时间分别变成 3、1、2 秒。预测并发耗时、完成顺序和结果列表顺序，再运行。这个差异比背诵“异步更快”更值得记住。

## 4. 协程与 Task：声明工作，安排工作

前面的 `prepare()` 可以继续复用。把 `compare_async.py` 中 `main()` 的函数体替换为下面内容：

```python
    first = asyncio.create_task(prepare("笔记", 1))
    second = asyncio.create_task(prepare("习题", 1))
    print("两项工作已安排")
    print(await first)
    print(await second)
```

两个任务已经在等待它们之前被安排，因此这段仍然能并发推进。可以将 Task 理解为“交给事件循环管理的协程”，它带有完成、结果和取消等状态。保留任务引用，并在所属流程结束前等待或妥善取消它们。

若只是启动同一组工作并等待全部结束，Python 3.11+ 可使用 `TaskGroup`。替换同一个 `main()` 的函数体：

```python
    async with asyncio.TaskGroup() as group:
        first = group.create_task(prepare("笔记", 1))
        second = group.create_task(prepare("习题", 1))
    print(first.result(), second.result())
```

离开 `async with` 块时会等待组内任务结束。某项任务发生普通异常时，TaskGroup 会取消其余任务、等待清理，再将错误作为异常组抛出。相比之下，`gather()` 默认传播首个异常，但不会因此自动取消所有其他任务。任务、结果顺序和取消细节见 [官方 Coroutines and Tasks 文档](https://docs.python.org/3/library/asyncio-task.html)。

## 5. 超时和取消：等不到结果怎么办

以下为独立完整脚本 `timeout_demo.py`：

```python
import asyncio


async def slow_job():
    try:
        await asyncio.sleep(10)
        return "完成"
    finally:
        print("执行清理")


async def main():
    try:
        async with asyncio.timeout(0.1):
            await slow_job()
    except TimeoutError:
        print("等待超时")


if __name__ == "__main__":
    asyncio.run(main())
```

预期先打印“执行清理”，再打印“等待超时”。超时通过取消机制打断等待，`finally` 中仍可清理资源。`TimeoutError` 在超时上下文之外捕获。

不要为了让日志安静而吞掉 `asyncio.CancelledError`；确实需要捕获它做清理时，清理后通常应重新抛出。取消是协作式请求，并不保证在任意一行立即终止工作。

## 6. 限制并发数量：别一次发出所有请求

以下完整脚本 `limited_async.py` 同时最多处理两份资料：

```python
import asyncio


async def main():
    semaphore = asyncio.Semaphore(2)
    active = 0

    async def prepare(index):
        nonlocal active
        async with semaphore:
            active += 1
            try:
                print(f"开始 {index}，正在处理 {active} 项")
                await asyncio.sleep(0.2)
                return index
            finally:
                active -= 1

    results = await asyncio.gather(*(prepare(i) for i in range(5)))
    print(results)


if __name__ == "__main__":
    asyncio.run(main())
```

检查日志：正在处理的数量不应超过 2，最后结果为 `[0, 1, 2, 3, 4]`。`async with` 会在离开区域时释放名额，即使区域内发生异常。参见 [官方同步原语文档](https://docs.python.org/3/library/asyncio-sync.html#semaphore)。

Semaphore 限制的是同时进入某段代码的数量，不是每秒请求数；这里仍会为全部输入安排工作。任务达到几十万条时，可以采用固定数量的消费者和有界队列，控制排队规模。

## 7. 如何收集结果与处理异常

### 7.1 gather 与 as_completed：结果顺序不同

`gather()` 适合“这一组任务结束后，按输入顺序拿到结果”。如果希望先完成的资料先展示，可以使用 `as_completed()`。为兼容 Python 3.11，本例采用普通 `for` 加 `await` 的形式。

保存并运行 `completion_order.py`：

```python
# completion_order.py
import asyncio


async def prepare(name, delay):
    await asyncio.sleep(delay)
    return name


async def main():
    tasks = [
        asyncio.create_task(prepare("A", 0.3)),
        asyncio.create_task(prepare("B", 0.1)),
        asyncio.create_task(prepare("C", 0.2)),
    ]
    for next_result in asyncio.as_completed(tasks):
        print("先收到：", await next_result)


if __name__ == "__main__":
    asyncio.run(main())
```

通常依次输出 B、C、A。所有任务已经创建，循环是在收结果，不是在每轮才启动一项工作。不要把普通迭代形式中的 `next_result` 当作原始 Task 的身份标识；需要关联输入时，可让协程返回 `(输入标识, 结果)`。

### 7.2 三种常见失败策略

| 策略 | 工具或写法 | 适合的场景 |
| --- | --- | --- |
| 同组任务共同成功或失败 | `TaskGroup` | 同一个操作的多个相关步骤 |
| 一项失败，仍收集其他结果 | `gather(..., return_exceptions=True)` | 一批彼此独立的资料 |
| 每项自己转成明确的成功/失败记录 | 在工作函数中捕获预期异常 | 需要展示每项失败原因的导入流程 |

`gather()` 默认抛出首个异常，并不等于“已经停止全部工作”。另外，如果这个异常导致顶层 `main()` 直接退出，`asyncio.run()` 清理剩余任务时又可能取消它们；要区分“gather 的行为”和“程序退出的清理行为”。

保存 `collect_failures.py`，观察一项失败时的结果：

```python
# collect_failures.py
import asyncio


async def prepare(index):
    await asyncio.sleep(0.05)
    if index == 2:
        raise ValueError("第 2 份资料格式错误")
    return f"资料 {index}"


async def main():
    results = await asyncio.gather(
        *(prepare(i) for i in [1, 2, 3]),
        return_exceptions=True,
    )
    for index, result in zip([1, 2, 3], results):
        if isinstance(result, BaseException):
            print(index, "失败", type(result).__name__)
        else:
            print(index, "成功", result)


if __name__ == "__main__":
    asyncio.run(main())
```

这里第 1、3 项成功，第 2 项失败。由于结果列表中可能包含异常对象，不能再无条件把每个元素当字符串使用。用 `BaseException` 识别还包括取消结果；这不是在捕获并吞掉整个程序的所有异常。

### 7.3 ExceptionGroup 与 except*

TaskGroup 中的普通任务异常会在退出时形成异常组（ExceptionGroup）；`except*` 用来按类型处理组内匹配的异常。一个异常组可能只包含一项，并不要求同时出两个错误。

保存并运行 `taskgroup_failure.py`：

```python
# taskgroup_failure.py
import asyncio


async def fail():
    await asyncio.sleep(0.05)
    raise ValueError("资料格式错误")


async def slow():
    try:
        await asyncio.sleep(5)
        print("慢任务正常完成")
    finally:
        print("慢任务执行清理")


async def main():
    try:
        async with asyncio.TaskGroup() as group:
            group.create_task(fail())
            group.create_task(slow())
    except* ValueError as errors:
        print("处理 ValueError 数量：", len(errors.exceptions))


if __name__ == "__main__":
    asyncio.run(main())
```

约 0.05 秒后会看到清理输出和数量 1，不应看到“慢任务正常完成”。`except* ValueError` 只处理匹配类型；若还有其他类型未处理，剩余异常会继续传播。同一 `try` 不能同时混用普通 `except` 和 `except*`。组内任务取消、结果收集的完整约定见 [Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html)，异常组语法见 [Python 语言参考](https://docs.python.org/3/reference/compound_stmts.html#except-star)。

## 8. 协程链与队列：两种组织工作的方式

### 8.1 协程链：同一条资料顺序处理，多条资料同时推进

资料处理需要两步：先获取目录信息，再根据目录找到练习。这两步有依赖，必须顺序执行；不同课程的两步流程可以并发。保存 `course_chain.py`：

```python
# course_chain.py
import asyncio
from time import perf_counter


async def fetch_outline(course_id):
    await asyncio.sleep(0.1)
    return {"course_id": course_id, "exercise_key": f"E-{course_id}"}


async def fetch_exercise(key):
    await asyncio.sleep(0.2)
    return f"练习 {key}"


async def load_course(course_id):
    outline = await fetch_outline(course_id)
    exercise = await fetch_exercise(outline["exercise_key"])
    return course_id, exercise


async def main():
    start = perf_counter()
    print(await asyncio.gather(*(load_course(i) for i in [1, 2, 3])))
    print(f"耗时约 {perf_counter() - start:.2f} 秒")


if __name__ == "__main__":
    asyncio.run(main())
```

每条链约 0.3 秒，三条并发仍约 0.3 秒；若逐条加载则约 0.9 秒。为什么不能把单条链的两个函数直接并列交给 gather？因为练习的 key 要等目录结果回来才知道。协程链（coroutine chain）只是把这种依赖显式写出来。

### 8.2 队列：工作不断到来，消费者按能力处理

若资料不断到达，不适合提前拿到全部输入再批量 `gather()`。可以由生产者（producer）负责投递，消费者（consumer）负责处理，中间用队列（queue）传递。

```text
生产者 → [有界队列：最多暂存 2 项] → 消费者 A
                                  → 消费者 B
```

保存并运行 `queue_workers.py`。本例把 0–4 作为资料编号，2 故意处理失败，仍继续处理其余资料：

```python
# queue_workers.py
import asyncio


async def producer(queue, worker_count):
    for index in range(5):
        await queue.put(index)
    for _ in range(worker_count):
        await queue.put(None)


async def worker(name, queue, results):
    while True:
        item = await queue.get()
        try:
            if item is None:
                return
            await asyncio.sleep(0.05)
            if item == 2:
                raise ValueError("资料不可用")
            results[item] = "成功"
            print(name, "处理", item)
        except ValueError as error:
            results[item] = str(error)
        finally:
            queue.task_done()


async def main():
    queue = asyncio.Queue(maxsize=2)
    results = {}
    async with asyncio.TaskGroup() as group:
        group.create_task(worker("A", queue, results))
        group.create_task(worker("B", queue, results))
        await producer(queue, worker_count=2)
        await queue.join()
    print(sorted(results.items()))


if __name__ == "__main__":
    asyncio.run(main())
```

最后输出应为：

```text
[(0, '成功'), (1, '成功'), (2, '资料不可用'), (3, '成功'), (4, '成功')]
```

逐个理解这里容易漏掉的环节：

1. 两个消费者先被安排执行，再开始投递。有界队列满时，`await put()` 会等待消费者腾出空间。
2. `maxsize=2` 只限制队列里等待领取的数量，不包括已经被消费者拿走的项目。两个消费者各处理一项时，还可以暂存两项。
3. `None` 是哨兵值（sentinel），约定为结束信号。消费者有两个，所以发送两个；此例的正常数据都是整数，不会与 None 混淆。
4. `task_done()` 表示这次领取的工作已被确认处理，不表示一定成功。`finally` 确保成功、预期失败和收到结束信号都正确计数。
5. `join()` 等待未完成计数归零，不会主动让无限循环的消费者退出。真正让它们退出的是结束信号，TaskGroup 再等待退出完成。

**背压（backpressure）**出现在生产者因为队列满而等待时。它把“下游处理不过来”传回上游，避免上游无限堆积数据。若只是限制同时请求的数量，用 Semaphore 更直接；若还要控制等待区大小和工作者数量，队列更合适。

本例只捕获可以按单项处理的 `ValueError`。若出现意外异常，TaskGroup 会让整个流程失败并取消其他任务，避免静默丢失消费者后无限等待。队列计数语义见 [Queues 文档](https://docs.python.org/3/library/asyncio-queue.html)。

## 9. async for、异步生成器和 async with

### 9.1 异步生成器：数据一批批来，不必全部攒齐

保存 `async_stream.py`：

```python
# async_stream.py
import asyncio


async def chapters():
    for number in range(1, 4):
        await asyncio.sleep(0.1)
        yield {"number": number, "title": f"第 {number} 章"}


async def main():
    async for chapter in chapters():
        print(chapter["title"])

    selected = [
        chapter["number"]
        async for chapter in chapters()
        if chapter["number"] >= 2
    ]
    print(selected)


if __name__ == "__main__":
    asyncio.run(main())
```

第一轮约每 0.1 秒出现一章；第二轮重新创建生成器并得到 `[2, 3]`。`async def` 中使用 `yield` 就定义了异步生成器（asynchronous generator）。调用它得到可以逐项异步读取的对象，而不是一个应当直接 `await` 的普通协程。

异步可迭代对象提供 `__aiter__()`；异步迭代器的 `__anext__()` 提供下一项的可等待操作，结束时使用 `StopAsyncIteration`。日常使用 `async for` 就好，暂时不用手写这些协议方法。

`async for` 并没有自动同时处理三章。它等待第一项、处理第一项，再取下一项；异步推导式（asynchronous comprehension）也是如此。并发要额外安排任务。若数据量大且想边读边处理，直接逐项消费即可，不必又用列表推导式把所有结果存入内存。

### 9.2 async with：进入与退出也可能要等待

普通 `with` 常用于文件关闭；异步上下文管理器（asynchronous context manager）让进入和退出阶段也可以使用 await，例如建立连接、提交收尾动作或释放远程资源。

保存 `async_resource.py`：

```python
# async_resource.py
import asyncio
from contextlib import asynccontextmanager


@asynccontextmanager
async def study_session():
    print("开始建立会话")
    await asyncio.sleep(0.05)
    try:
        yield {"status": "ready"}
    finally:
        await asyncio.sleep(0.05)
        print("会话已释放")


async def main():
    try:
        async with study_session() as session:
            print(session["status"])
            raise ValueError("模拟处理失败")
    except ValueError:
        print("调用方收到错误")


if __name__ == "__main__":
    asyncio.run(main())
```

输出依次是建立会话、ready、释放会话、收到错误。`yield` 之前相当于进入，yield 出来的对象交给 `as session`，`finally` 负责离开时的清理。清理逻辑执行不意味着业务错误被吞掉。

底层协议是 `__aenter__()` 和 `__aexit__()`；`@asynccontextmanager` 帮助我们用生成器形式表达它。这个例子不建立真实网络连接，目的是单独观察生命周期。详见 [contextlib 官方文档](https://docs.python.org/3/library/contextlib.html#contextlib.asynccontextmanager)。

### 9.3 从模拟等待走向真实 HTTP 请求

实际请求应使用支持异步的客户端。原文展示 aiohttp；下面使用同样支持异步的 HTTPX，便于共享一个客户端并设置超时。学习此例需要 HTTPX：

```bash
python -m pip install httpx
```

保存 `http_checks.py`。默认请求两个公开文档首页，也可在运行时传入自己的测试 URL：

```python
# http_checks.py
import asyncio
import sys
import httpx


async def check(client, url):
    try:
        response = await client.get(url)
        response.raise_for_status()
        return {"url": url, "status": response.status_code}
    except httpx.HTTPStatusError as error:
        return {"url": url, "error": f"HTTP {error.response.status_code}"}
    except httpx.RequestError as error:
        return {"url": url, "error": type(error).__name__}


async def main(urls):
    limits = httpx.Limits(max_connections=2)
    async with httpx.AsyncClient(
        timeout=5.0, limits=limits, follow_redirects=True,
    ) as client:
        results = await asyncio.gather(*(check(client, url) for url in urls))
    for result in results:
        print(result)


if __name__ == "__main__":
    urls = sys.argv[1:] or ["https://www.python.org/", "https://docs.python.org/3/"]
    asyncio.run(main(urls))
```

建立一个共享客户端，可以复用连接池（connection pool）；不要为了每个小请求都重新创建客户端。5 秒的设置用于 HTTPX 的各类网络超时，并不是整个批处理一定在 5 秒内结束的保证。若需要给整个工作设置总时限，另外使用 `asyncio.timeout()`。

HTTP 状态失败（例如 404）表示已收到服务端响应，但状态不符合成功要求；连接失败或读取超时则属于请求层面的错误。HTTP 200 也不保证业务内容正确，下一节的综合案例还会做数据验证。公网结果受网络和站点状态影响，因此此例不承诺固定状态码。用法见 [HTTPX Async Support](https://www.python-httpx.org/async/) 和 [Timeouts](https://www.python-httpx.org/advanced/timeouts/)。

### 其他概念索引

| 概念 | 在实际程序中的用途 | 容易误解的地方 |
| --- | --- | --- |
| 协程链 | 同一份资料先下载、再解析；不同资料之间并发 | 有依赖的步骤仍需要按顺序等待 |
| `asyncio.Queue` | 生产者放任务，固定数量的消费者处理 | 每次成功取出的工作需对应 `task_done()`；`join()` 等待全部被确认处理 |
| `async for` | 读取异步迭代器，如分批返回的数据流 | 给普通循环加 `async` 不会自动让各轮并发 |
| `async with` | 管理需要异步进入或退出的资源 | 常用于连接、锁和客户端的生命周期 |

队列是生产者与消费者之间的缓冲区。设置 `maxsize` 后，队列满时 `await put()` 会等待，让生产速度受到约束。`queue.join()` 不会自动关闭无限循环的消费者，程序还需要结束信号或取消策略。详见 [官方 Queues 文档](https://docs.python.org/3/library/asyncio-queue.html)。异步迭代和上下文管理协议见 [Python 语言参考](https://docs.python.org/3/reference/compound_stmts.html#coroutines)。

## 10. 阻塞操作与共享状态：两个看起来正常的陷阱

### 10.1 用小实验看见阻塞

保存 `blocking_demo.py`：

```python
# blocking_demo.py
import asyncio
import time


def old_reader():
    time.sleep(0.2)
    return "读完"


async def heartbeat():
    for _ in range(3):
        print("心跳", flush=True)
        await asyncio.sleep(0.05)


async def read_bad():
    print(old_reader())


async def read_good():
    print(await asyncio.to_thread(old_reader))


async def main():
    print("直接调用同步函数：")
    await asyncio.gather(read_bad(), heartbeat())
    print("交给线程：")
    await asyncio.gather(read_good(), heartbeat())


if __name__ == "__main__":
    asyncio.run(main())
```

第一段会先卡住约 0.2 秒，打印“读完”后才开始心跳；第二段等待同步函数期间，心跳可以推进。观察重点是响应性，不只是总时间。`to_thread()` 并未把同步函数内部改造成异步，而是把它移到其他线程，调用方异步等待结果。它也不保证停止等待时那个线程会马上停止。

### 10.2 单线程也可能发生竞态条件

竞态条件（race condition）是结果依赖多个操作的交错顺序。假设两个任务都先读出计数 0，接着都等待，再各自写回 1，最终就丢了一次增长。

保存 `shared_counter.py`：

```python
# shared_counter.py
import asyncio


async def run_trial(use_lock):
    counter = 0
    lock = asyncio.Lock()

    async def change():
        nonlocal counter
        before = counter
        await asyncio.sleep(0)
        counter = before + 1

    async def worker():
        if use_lock:
            async with lock:
                await change()
        else:
            await change()

    await asyncio.gather(worker(), worker())
    return counter


async def main():
    print("无锁：", await run_trial(False))
    print("有锁：", await run_trial(True))


if __name__ == "__main__":
    asyncio.run(main())
```

在默认任务调度下，这个有意插入让出点的实验会输出无锁 1、有锁 2。锁（lock）保护的是“读取 → 等待 → 写回”的整段逻辑。真实计数器若根本不需要跨 await，可优先让更新在一个连续步骤内完成，或者将状态交给一个消费者统一维护。异步锁也不是跨进程或跨线程的通用锁。详见 [Synchronization Primitives](https://docs.python.org/3/library/asyncio-sync.html)。

### 常见错误回查

**只写 `greet()`，忘记运行它。** 你得到的是协程对象；如果一直不等待它，会看到“coroutine was never awaited”之类的警告。脚本入口用 `asyncio.run()`，协程内部用 `await` 或安排任务。

**在异步函数中调用 `time.sleep()`。** 它会阻塞事件循环所在的线程。模拟异步等待用 `await asyncio.sleep()`。同步 HTTP 调用和普通文件读写也不会因为放进 `async def` 就变成非阻塞操作。

**计算任务加上 async，期望自动加速。** 没有可让出的等待，长计算仍会占住事件循环。纯 Python CPU 密集任务通常需要考虑进程池等方案，而不是只改函数声明。

**以为只要单线程就没有共享状态问题。** 一段“读状态 → await → 写状态”之间，其他任务可能改动状态；跨等待点维护共享数据时，仍需考虑锁和一致性。

遇到已有同步 I/O 函数，可以研究 `await asyncio.to_thread(sync_function, argument)`，将它放在线程中执行，避免直接阻塞事件循环；取消等待通常不会强行停止那个线程里的函数。调试阻塞和并发问题可读 [官方 Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)。

## 11. 自测：先预测，再运行

1. 连续写两句 `await prepare("A", 1)` 和 `await prepare("B", 1)`，约需几秒？
2. 先 `create_task()` 创建 A、B 两个任务，然后依次等待它们，约需几秒？
3. `gather(A, B)` 中 B 先完成，结果列表会把 B 放前面吗？
4. Semaphore 的值为 2，能否保证每秒最多两个请求？

<details markdown="1">
<summary>完成后展开参考答案</summary>

1. 约两秒，两项工作依次执行。
2. 约一秒，两个任务已提前安排。
3. 不会，结果按传入顺序排列。
4. 不能，它约束同时进行的数量。每秒次数需要另外的速率控制。

</details>

最后做一个小改造：把并发对比实验中的三份资料改成五份，每份等待 0.5 秒，再将并发上限设为 2。预计需要三批，等待时间约 1.5 秒。用日志确认每批最多两项，而不是只看总用时。

## 12. 综合实践：并发获取，再逐条验证

两篇笔记可以形成一条工作流程：**并发获取原始数据 → 用 Pydantic 校验每条记录 → 只让成功的记录进入后续处理。** `asyncio` 管任务如何推进，Pydantic 管数据是否符合规则；Pydantic 的普通模型验证仍是同步工作。

下面把这个流程真正写成可运行程序。保存 `validated_fetch.py`，需要 Pydantic 2。为了让任何时候都能复现实验，`fetch_raw()` 使用本地模拟数据，不访问外部服务。

```python
# validated_fetch.py
import asyncio
from pydantic import BaseModel, Field, ValidationError


class Lesson(BaseModel):
    title: str = Field(min_length=1)
    minutes: int = Field(gt=0)


async def fetch_raw(index):
    await asyncio.sleep(0.05)
    if index == 3:
        raise ConnectionError("模拟连接失败")
    if index == 2:
        return {"title": "", "minutes": -10}
    return {"title": f"课程 {index}", "minutes": "30"}


async def load_one(index, semaphore):
    try:
        async with semaphore:
            async with asyncio.timeout(1):
                raw = await fetch_raw(index)
    except (ConnectionError, TimeoutError) as error:
        return {"id": index, "ok": False, "stage": "fetch",
                "error": type(error).__name__}

    try:
        lesson = Lesson.model_validate(raw)
    except ValidationError as error:
        return {"id": index, "ok": False, "stage": "validation",
                "fields": [list(item["loc"]) for item in error.errors()]}

    return {"id": index, "ok": True, "data": lesson.model_dump()}


async def main():
    semaphore = asyncio.Semaphore(2)
    results = await asyncio.gather(*(load_one(i, semaphore) for i in [1, 2, 3]))
    for result in results:
        print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

预期第 1 条成功，分钟数转成整数 30；第 2 条在 validation 阶段失败，标出 title 和 minutes；第 3 条在 fetch 阶段失败，原因是 ConnectionError。每条结果保留输入编号，便于用户知道哪项需要修复或重试。

这里把超时放在取得 Semaphore 名额之后，所以一秒时限不包含排队等待名额的时间。若业务要求“从开始排队到获取完成最多一秒”，应把 timeout 放在 semaphore 外层。**同样的工具，嵌套位置不同，业务含义就不同。**

将来替换成 HTTPX 时，先检查 HTTP 状态并解析 JSON，再进入模型验证。真实客户端抛出的是对应的 `httpx.RequestError` 等异常，要随实现调整捕获类型；别假定所有网络库都使用这里模拟的 `ConnectionError`。JSON 格式错误也属于需要单独辨认的数据解析失败。

## 13. 实践作业与参考思路

1. 把第 1 条的等待改成两秒，确认它被标记为超时，其他项仍有结果。
2. 增加第 4 条正确数据，验证并发上限仍是 2。
3. 在任务开始和结束处打印带编号的日志，解释为什么结果列表顺序与完成顺序不一定一致。
4. 想让任何一条非法数据都使整批失败，应如何改变目前“每条返回错误记录”的策略？

<details markdown="1">
<summary>展开参考思路</summary>

第 1 题在 `fetch_raw()` 对 index 1 等待两秒；外层会得到 TimeoutError，同时仍会执行异步清理。第 2 题将输入列表改成 `[1, 2, 3, 4]`，模拟函数会为 4 返回正确记录。第 3 题关注日志反映执行进度，gather 输出反映输入位置。第 4 题可让验证异常继续传播，并将相关任务纳入 TaskGroup；同时设计整批成功后才提交结果的步骤，避免部分任务已产生不可撤销的副作用。

</details>

## 14. 查问题时按这个顺序

先检查是否在正确的 Python 环境，其次看是没 await、嵌套事件循环，还是同步操作阻塞；再确认任务究竟何时被安排、是否仍有未结束的工作，最后区分网络错误、数据错误和取消。

在终端中可以用 `PYTHONASYNCIODEBUG=1 python your_script.py` 开启调试模式，帮助发现遗漏等待或耗时回调等问题。调试模式会增加开销，适合排错。使用方式见 [Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)。

继续学习时，按实际需求选择下一项：HTTP 客户端、数据库的异步驱动，或基于 ASGI（Asynchronous Server Gateway Interface，异步服务器网关接口）的 Web 框架。先把这篇的生命周期、错误处理和数量控制练熟，再接入更多库。
