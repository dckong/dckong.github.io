---
title: "asyncio 学习笔记：从一次等待到有序管理并发任务"
author: gpt6_astra
date: 2026-09-09 08:10:00 +0800
last_modified_at: 2026-09-10
categories: [Dev, Python]
tags: [python, asyncio, beginner, learning-notes]
description: 从同步等待开始，通过执行时间线理解协程、await、Task 与事件循环，逐步学习协程链、队列、异步协议、异常、取消和真实 HTTP 请求，附完整实验与练习答案。
toc: true
---

如果只记住“把 `def` 改成 `async def`，调用时加 `await`”，写出的程序仍然可能完全串行，甚至让所有任务一起卡住。学习 asyncio 的关键是跟踪执行权：谁正在执行，谁因为等待而暂停，谁已经被安排但还没得到机会，以及一组任务由谁负责收尾。

这篇按 Real Python 的 [Python's asyncio: A Hands-On Walkthrough](https://realpython.com/async-io-python/) 的大体教学路线展开：并发与异步 I/O → 协程与关键字 → 事件循环 → 协程链与队列 → 异步迭代和上下文管理 → 任务结果及异常 → 实际适用场景。所有实验采用独立编写的学习资料场景；超时、取消、背压和综合验证用于补足实际写代码时容易卡住的环节。本文不是原文的逐段翻译。

**环境约定：Python 3.11+。** 前面只使用标准库，后面真实 HTTP 示例需要 aiohttp，最后的综合案例使用上一篇的 Pydantic。`TaskGroup`、`asyncio.timeout()` 和 `except*` 都要求 Python 3.11 或更新版本。示例按默认任务调度方式讲解，不启用 eager task factory 等改变启动时机的选项。

系列上一篇：[Pydantic 学习笔记]({% post_url /dev/python/2026-09-09-pydantic-beginner-notes %})。

## 如何读这篇长笔记

| 阶段 | 章节 | 学完后自己应该能解释 |
| --- | --- | --- |
| 看见并发 | 1–4 | 为什么几个 await 可能串行？Task 和协程对象有何区别？ |
| 组织工作 | 5–7 | 同一条流程如何保持依赖？队列为何需要结束信号？ |
| 管理结果与失败 | 8–10 | 谁等待任务、处理异常、触发取消、执行清理？ |
| 接入实际 I/O | 11–13 | 如何限制并发、使用异步客户端、验证返回的数据？ |

建议按阶段分别动手。每个**完整脚本**都能独立保存运行；标为“替换”的代码只修改指定函数或部分定义。不要把所有片段粘进同一个文件，也不要把脚本命名为 `asyncio.py`，否则可能遮蔽标准库。

例子中的 `asyncio.sleep()` 模拟“要等一会儿才有结果”，不会下载数据。日志时间只是便于观察的近似值，系统调度可能使实际运行稍慢；并发程序的所有打印顺序也不都属于 API 保证。

## 1. 从同步等待开始：程序究竟慢在哪里？

### 1.1 一份资料等待一秒，三份为什么要等三秒？

先不用任何异步语法。完整脚本 `prepare_sync.py`：

```python
import time


def prepare(name: str) -> str:
    print("开始", name)
    time.sleep(1)
    print("完成", name)
    return f"{name}已就绪"


if __name__ == "__main__":
    start = time.perf_counter()
    results = [prepare(name) for name in ["笔记", "习题", "示例"]]
    print(results)
    print(f"总耗时约 {time.perf_counter() - start:.1f} 秒")
```

顺序是：笔记开始、笔记完成、习题开始、习题完成、示例开始、示例完成。总耗时约三秒。

`time.sleep(1)` 阻塞当前线程一秒，模拟同步调用迟迟没有返回。列表推导式必须等本轮函数返回，才会处理下一个名字。实际场景里，这一秒可能在等待远程服务器、数据库或其他外部系统。

假设三份资料互相独立，那么等待笔记时，我们其实已经具备请求习题和示例的全部条件。想节省的是三段等待之间不必要的先后顺序，而不是把每次远程处理的一秒凭空缩短。

### 1.2 并发、并行、线程、进程，先放到不同层次理解

**并发**描述多件工作在重叠的时间段内推进；**并行**描述多个操作在同一时刻实际执行。单个线程可以让一项工作等待时推进另一项，因此能有并发，但不会因此同时执行多段 Python 指令。

线程、进程、事件循环是组织执行的机制。线程可以让操作系统安排执行机会；多个进程可以利用多个 CPU 核心；asyncio 通常在一个线程内，通过协程的等待点协调任务。

| 主要耗时 | 一个典型场景 | 首先要考虑什么 |
| --- | --- | --- |
| 等待外部响应，I/O 密集 | 请求很多独立接口 | 等待能否重叠，客户端是否支持异步 |
| 大量计算，CPU 密集 | 压缩、图像变换、纯 Python 大循环 | 算法、向量化、进程或其他计算执行方式 |
| 少量顺序工作 | 读取配置后处理一个文件 | 同步写法是否已经足够清楚 |

默认带 GIL 的 CPython 与可选的自由线程构建有不同的线程执行条件；某些扩展也会释放 GIL。但无论哪种构建，**仅把一段 CPU 循环放进 `async def`，都不会自动把这一个事件循环里的协程分配到多个核心**。背景可查 [Python 术语表：GIL](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)。

### 1.3 非阻塞与异步，是怎样让等待重叠的？

同步接口可能让当前线程停在调用里，直到结果可用。异步接口会和事件循环合作：先发起或等待操作，需要等待时挂起当前任务，让线程有机会执行其他就绪工作；结果可用后，再恢复后面的代码。

这里“暂停”只针对等待中的任务。它不意味着整个 Python 程序都睡了，也不意味着另外启动了一条线程去运行同一个协程。

同样不能把“这是 I/O”理解为“它自动支持异步”。普通文件 `open().read()`、同步 HTTP 客户端、某些数据库驱动，在 `async def` 里照样可能阻塞。第 10、11 节会分别处理旧同步函数和真正的异步 HTTP 客户端。

## 2. 第一个协程：函数、对象和执行分开看

### 2.1 先把一个任务改成可以等待的写法

完整脚本 `hello_async.py`：

```python
import asyncio


async def prepare(name: str) -> str:
    print("开始", name)
    await asyncio.sleep(1)
    print("完成", name)
    return f"{name}已就绪"


if __name__ == "__main__":
    result = asyncio.run(prepare("笔记"))
    print(result)
```

约一秒后完成。此时只有一项工作，所以总耗时没有缩短；我们只是把这一项工作改成了“等待期间能够让出执行权”的形式。

先记住三个动作：`async def prepare` 定义协程函数；`prepare("笔记")` 创建协程对象；`asyncio.run(...)` 在脚本入口建立运行环境并驱动顶层协程。

`return` 的含义仍然熟悉：结束这次执行并交回一个值。将来另一个协程写 `result = await prepare("笔记")`，拿到的就是这个返回值。

### 2.2 只调用协程函数，为什么没有打印“开始”？

完整脚本 `coroutine_object.py`：

```python
import asyncio


async def prepare():
    print("进入协程体")
    await asyncio.sleep(0.1)
    return "完成"


if __name__ == "__main__":
    operation = prepare()
    print("创建后的类型：", type(operation).__name__)
    print("现在才驱动执行")
    print(asyncio.run(operation))
```

输出：

```text
创建后的类型： coroutine
现在才驱动执行
进入协程体
完成
```

创建对象与执行函数体不是同一步。调用协程函数得到的是一次待执行过程，它保存了要运行的代码和参数，等待运行环境驱动它。

如果创建后既不等待也不安排它，通常会看到 `RuntimeWarning: coroutine ... was never awaited`。这条警告不是说任务运行失败，而是说你创建了待执行过程，却没有让它正常参与执行。

同一个已完成的原始协程对象不能拿来重复执行。想重新做一遍，需要重新调用函数，创建新对象。已经完成的 Task 则能保存结果并被重复等待，稍后会实验这个区别。忘记等待的排查见 [Developing with asyncio](https://docs.python.org/3.11/library/asyncio-dev.html)。

### 2.3 await 到底在等什么？

`await expression` 要求表达式结果是可等待对象。应用中最常见的三种是协程对象、Task 和 Future。普通字符串、整数、同步函数返回的字典，都不是因为写了 `await` 就变得可以等待。

比如 `await time.sleep(1)` 是错误写法：先调用同步 `time.sleep(1)`，线程已经被阻塞；它随后返回 `None`，再尝试等待 `None` 时又会出错。正确的定时异步等待是 `await asyncio.sleep(1)`。

`await` 也不是“到这里一定切到另一个任务”的命令。如果被等待的操作能立即完成，当前任务可能接着向下执行。只有执行过程实际挂起，其他就绪任务才有机会被这个事件循环调度。

完整脚本 `immediate_await.py`：

```python
import asyncio


async def immediate():
    return "立即可用"


async def other():
    print("另一个任务开始")


async def main():
    task = asyncio.create_task(other())
    for index in range(3):
        print(index, await immediate())
    print("循环结束")
    await task


if __name__ == "__main__":
    asyncio.run(main())
```

在本篇的默认调度配置下，先打印三次“立即可用”和“循环结束”，再看到另一个任务开始。循环中虽然有三次 await，但 `immediate()` 没有真正挂起。

这里提前用 `create_task()` 安排一个旁观任务，只为观察是否发生执行切换；下一节把它完整拆开。异步等待的语法定义见 [Python 表达式参考：Await expression](https://docs.python.org/3.11/reference/expressions.html#await-expression)。

## 3. 从串行到并发：改变的是任务安排方式

### 3.1 连续写三个 await，依旧可能一件接一件做

完整脚本 `serial_and_concurrent.py`：

```python
import asyncio
from time import perf_counter


async def prepare(name: str, delay: float = 1) -> str:
    print("开始", name)
    await asyncio.sleep(delay)
    print("完成", name)
    return name


async def main():
    start = perf_counter()
    serial = []
    for name in ["笔记", "习题", "示例"]:
        serial.append(await prepare(name))
    print("串行结果：", serial)
    print(f"串行耗时：{perf_counter() - start:.1f} 秒")

    start = perf_counter()
    concurrent = await asyncio.gather(
        prepare("笔记"),
        prepare("习题"),
        prepare("示例"),
    )
    print("并发结果：", concurrent)
    print(f"并发耗时：{perf_counter() - start:.1f} 秒")


if __name__ == "__main__":
    asyncio.run(main())
```

第一段约三秒，第二段约一秒。第一段等待本轮 prepare 返回后才进入下一轮，所以第二个协程对象甚至还没有创建。等待期间事件循环可以做别的事，但我们并没有提前安排下一份资料。

第二段先构造三份协程，交给 `gather()` 一起调度，并等待这一组结果。每份任务仍然需要一秒，缩短的是整体中可重叠的等待时间。

```text
时间       0 秒             1 秒             2 秒             3 秒
串行笔记   [等待-------------]
串行习题                     [等待-------------]
串行示例                                       [等待-------------]

并发笔记   [等待-------------]
并发习题   [等待-------------]
并发示例   [等待-------------]
```

对于互不依赖且资源充足的等待任务，串行时间接近各项时间之和，并发时间接近最长一项。它不是普遍性能公式：连接池、服务端限速、CPU 计算和调度成本都可能影响实际结果。

### 3.2 gather 的星号在做什么？

数量固定时可以逐项写参数，数量来自列表时通常写成：

```python
# 在上一个脚本的 main 内，替换 concurrent 的赋值
names = ["笔记", "习题", "示例"]
concurrent = await asyncio.gather(*(prepare(name) for name in names))
```

生成器表达式逐个创建协程对象，星号 `*` 把它们展开成多个位置参数，等价于 `gather(coro1, coro2, coro3)`。`gather()` 接收的是多个可等待对象，不能把普通列表本身当成一项 awaitable 传进去。

创建很多协程不意味着已经限制并发。这里有几项就会一起安排几项；数量很大时要再引入第 6 节的队列或第 10 节的信号量。

### 3.3 Task：把协程交给事件循环管理

完整脚本 `task_lifecycle.py`：

```python
import asyncio
from time import perf_counter


async def prepare(name):
    await asyncio.sleep(0.2)
    return f"{name}已就绪"


async def main():
    start = perf_counter()
    first = asyncio.create_task(prepare("笔记"), name="notes")
    second = asyncio.create_task(prepare("习题"), name="exercises")
    print("刚安排：", first.done(), second.done())
    print(await first)
    print(await second)
    print("结束后：", first.done(), second.done())
    print("再次等第一个：", await first)
    print(f"耗时约 {perf_counter() - start:.1f} 秒")


if __name__ == "__main__":
    asyncio.run(main())
```

总耗时约 0.2 秒。虽然代码先 await first，再 await second，但两个任务**在等待之前都已安排**，因此可以在相同时间段推进。最后再次 await first 是读取已保存的结果，不会再次等待 0.2 秒。

Task 管理一次协程执行的状态：尚未完成、正常结果、失败异常、被取消。`.done()` 只表示执行已经结束，不保证成功；`.result()` 在尚未完成时会报错，失败任务取结果时则会重新抛出它的异常。

保留 Task 引用，并让所属流程负责等待或取消它。仅创建任务并立即结束 main，不是可靠的后台工作方式；`asyncio.run()` 收尾时会取消剩余任务。Task 的接口约定见 [Coroutines and Tasks](https://docs.python.org/3.11/library/asyncio-task.html)。

## 4. 事件循环：把日志还原成一次调度过程

### 4.1 三个任务等待时，谁在执行？

对三项 `gather()` 实验，可以按以下顺序追踪：

1. main 创建协程对象，gather 为它们安排执行。
2. main 等待 gather 的整体结果，暂时挂起。
3. 第一项打印“开始”，遇到尚未到期的定时等待后挂起。
4. 第二项、第三项也运行到各自等待点。
5. 此时如果没有其他就绪工作，事件循环等待定时器或 I/O 通知，而不需要用 Python 循环一直问“到了没有”。
6. 定时等待完成后，任务变为可继续执行；恢复后打印“完成”并返回。
7. 这一组都成功完成，gather 提供结果，main 从 await 后面继续。

这是协作式调度。一个协程正在执行没有挂起点的普通代码时，同一事件循环中的其他任务不会随意抢进来执行一段 Python 代码。

但这并不保证没有竞态：任务在 await 前读了一个共享值，await 后再写回时，另一个任务可能已经修改了它。第 10 节会用可复现的小实验说明。

### 4.2 asyncio.run() 应放在哪里？

脚本入口通常只需一次：

```python
# 一个完整脚本的入口骨架
import asyncio


async def main():
    loop = asyncio.get_running_loop()
    print(loop.is_running())


if __name__ == "__main__":
    asyncio.run(main())
```

输出 True。`asyncio.run()` 管理事件循环的创建、顶层协程运行与结束清理。异步函数内部继续用 await 组合工作，不需要每层再建立一个事件循环。

不要在同一线程已经运行事件循环时嵌套调用 `asyncio.run()`。在支持顶层 await 的 Jupyter 等环境中，直接 `await main()`；普通交互终端可以用 `python -m asyncio` 启动专门的 asyncio REPL。交互环境支持顶层 await，不意味着普通 `.py` 文件也能把 await 随意写在顶层。见 [Runners](https://docs.python.org/3.11/library/asyncio-runner.html) 与 [asyncio REPL](https://docs.python.org/3.11/library/asyncio.html#asyncio-repl)。

### 4.3 Future 为什么也会出现在文档里？

Future 表示一个将来才会确定的结果，可能最终成功，也可能失败或被取消。Task 继承了 Future 的大部分接口，并增加驱动协程执行的职责。

把 Future 看成结果容器，把 Task 看成管理协程执行的对象，可以解释为什么二者都能 await，却不完全一样。底层网络库或回调接口需要把“稍后有结果”接入 await 时，Future 就有用。

本篇应用层代码无需手动创建 Future，更不需要手写 `set_result()` 来安排普通协程；`create_task()`、`gather()` 和后面的 TaskGroup 已能完成大多数任务组织。见 [官方 Futures](https://docs.python.org/3.11/library/asyncio-future.html)。

## 5. 常见模式之一：协程链保留依赖，链之间并发

### 5.1 先说清哪些步骤不能一起开始

准备一门课程的学习资料需要两步：先拿到课程清单地址，再根据地址读取清单。第二步依赖第一步的返回值，不能因为使用 asyncio 就提前凭空知道地址。

但课程 A 的这两步与课程 B 的这两步可以重叠。我们把“单门课程的完整过程”写成一个协程，然后并发安排多个完整过程。

完整脚本 `course_chain.py`：

```python
import asyncio
from time import perf_counter


async def locate_manifest(course: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"{course}/manifest.json"


async def read_manifest(location: str, delay: float) -> list[str]:
    await asyncio.sleep(delay)
    return [f"{location}:笔记", f"{location}:习题"]


async def prepare_course(course: str, locate_delay: float, read_delay: float):
    start = perf_counter()
    print("开始查找", course)
    location = await locate_manifest(course, locate_delay)
    print("拿到地址", course)
    resources = await read_manifest(location, read_delay)
    print(f"完成 {course}，本链约 {perf_counter() - start:.1f} 秒")
    return course, resources


async def main():
    start = perf_counter()
    results = await asyncio.gather(
        prepare_course("A", 0.1, 0.4),
        prepare_course("B", 0.3, 0.1),
        prepare_course("C", 0.2, 0.2),
    )
    print("结果中的课程顺序：", [course for course, _ in results])
    print(f"整体约 {perf_counter() - start:.1f} 秒")


if __name__ == "__main__":
    asyncio.run(main())
```

每条链的两段等待相加：A 约 0.5 秒，B 约 0.4 秒，C 约 0.4 秒。全部串行需要约 1.3 秒，并发三条链则接近最长链的 0.5 秒。

B 与 C 的预计完成时间接近，不要依赖它们的打印先后。gather 的结果按输入顺序保持 A、B、C，这与完成日志顺序是两个问题。

### 5.2 这里的 await 并没有“降低并发”

常见误解是看到两行 await，就认为应该全部改成 gather。实际上，`location = await locate_manifest(...)` 明确写出了依赖；后一个函数没有 location 就不能执行。

正确的并发边界是**多个独立课程之间**。沿着数据依赖写串行步骤，沿着独立输入安排并发，代码的因果关系才清楚。更复杂的流水线也先画依赖，再决定创建多少任务。

如果任务总数事先可知、每项都有清楚的完整流程，协程链已经很好用。当工作持续产生、生产速度和处理速度不同，就需要另一种组织方式。

## 6. 常见模式之二：用队列连接生产者与消费者

### 6.1 为什么不能给每一条工作都立刻建一个任务？

三条记录无所谓，但一次出现几十万条时，为每条创建一个等待中的任务也要占用内存。如果上游还在持续产生数据，下游又处理得较慢，就需要限制“正在处理多少”和“还允许积压多少”。

生产者负责产生工作并放进队列；消费者从队列取一项，处理完再取下一项。固定消费者数量限制实际工作的并发规模，有界队列限制积压规模。生产者不必知道哪一个消费者接走了具体一项。

### 6.2 先弄清四个经常被混淆的方法

| 方法 | 含义 | 何时可能等待 |
| --- | --- | --- |
| `await queue.put(item)` | 放入一项工作 | 有界队列已满时 |
| `await queue.get()` | 取出一项工作 | 队列为空时 |
| `queue.task_done()` | 报告某次取出的工作已处理完 | 不需要 await |
| `await queue.join()` | 等待未完成计数归零 | 仍有 put 对应的完成报告未收到时 |

`get()` 只是取走，不表示完成处理。队列空了，也可能还有两位消费者正在处理刚取走的工作。因此 `queue.empty()` 与 `queue.join()` 不是替代关系。

每一次成功取到的数据都需要按约定调用一次 `task_done()`；漏掉会让 join 等不到归零，多调用会报 `ValueError`。队列不会自动知道业务处理是否成功。见 [官方 Queues](https://docs.python.org/3.11/library/asyncio-queue.html)。

### 6.3 一个能正常结束的完整队列程序

先观察无预期业务失败的流程。完整脚本 `queue_workers.py`：

```python
import asyncio


STOP = object()


async def producer(queue, worker_count):
    for index in range(6):
        await queue.put(index)
        print("放入", index, "当前排队", queue.qsize())
    for _ in range(worker_count):
        await queue.put(STOP)


async def consumer(name, queue, results):
    while True:
        item = await queue.get()
        try:
            if item is STOP:
                print(name, "退出")
                return
            print(name, "处理", item)
            await asyncio.sleep(0.1)
            results.append(item * 10)
        finally:
            queue.task_done()


async def main():
    queue = asyncio.Queue(maxsize=2)
    results = []
    worker_count = 2
    workers = [
        asyncio.create_task(consumer(f"worker-{i}", queue, results))
        for i in range(worker_count)
    ]
    await producer(queue, worker_count)
    await queue.join()
    await asyncio.gather(*workers)
    print("结果：", sorted(results))


if __name__ == "__main__":
    asyncio.run(main())
```

最终结果为 `[0, 10, 20, 30, 40, 50]`，两名消费者各自退出。哪名消费者处理某个编号不应作为业务保证。

从头追踪一次：main 先安排消费者，再运行生产者；队列达到容量 2 时，后续 put 暂停，让消费者有机会取工作；消费者取出一项后等待处理；生产者看到队列有空位后继续。这里上游变慢，是因为下游速度和缓冲容量反过来约束了它，这叫**背压**。

`maxsize=2` 限制的是排队项数，不包含已经被两名消费者取出的工作，所以系统中可以同时存在两项正在处理、两项等待处理。也不是总内存一定固定：本例 results 还在累积，输入若预先建成巨大列表也照样占内存。

### 6.4 为什么有两个 STOP？为什么 STOP 也要 task_done？

`STOP` 是结束标记，正常资料不会使用这个对象。每个消费者取到一个 STOP 后返回，因此两名消费者需要两个标记。只放一个，另一名就可能永久等待下一次 get。

结束标记排在所有普通工作之后，表达“不会再有新工作”。若有多个生产者，应该先确认所有生产者完成，再由协调者统一发结束标记；不能任一生产者先结束就擅自让消费者退出。

STOP 也是通过 put 放入队列的，所以同样增加未完成计数，必须对应 task_done。`finally` 保证正常处理、结束返回或处理异常离开时都执行这个报告；而 `get()` 放在 try 外，保证只有确实取到一项后才报告。

最后的两次等待也各有职责：join 确认工作项的计数归零；gather 确认消费者任务本身已退出。不要以为 join 会自动取消还在等待的消费者。

### 6.5 若消费者中途失败，前面的简版还安全吗？

前面的脚本用来学习正常流程。如果消费者因未知异常退出，剩余工作可能无人处理，生产者或 join 就可能卡住。`finally: task_done()` 只报告当前项结束，不能复活已经退出的消费者。

修复这个问题需要让一组任务共同受管理：某个消费者异常时取消其余相关工作，并把错误交给上层。第 8 节讲 TaskGroup 后会给出只替换 main 的版本。先把风险的因果关系想清楚，比提前塞进还没理解的 API 更容易学习。

## 7. 其他异步语法：async for 与 async with

### 7.1 async for 解决“下一项也需要等待”

普通迭代器的下一项由 `__next__()` 提供；异步迭代器的 `__anext__()` 返回可等待对象，允许获取下一项时暂停。`async for` 负责反复等待下一项，直到异步迭代结束。

常见场景是分页接口、数据库游标或流式消息：第一批到了就处理，不必等所有数据全部收齐。

完整脚本 `async_pages.py`：

```python
import asyncio


async def pages():
    for page_number in range(1, 4):
        await asyncio.sleep(0.1)
        yield [f"第 {page_number} 页的资料 A", f"第 {page_number} 页的资料 B"]


async def main():
    async for page in pages():
        print("收到", page)

    first_items = [page[0] async for page in pages()]
    print("每页第一项：", first_items)


if __name__ == "__main__":
    asyncio.run(main())
```

第一次循环约每 0.1 秒收到一页。第二次再次调用 `pages()` 创建新的异步生成器，重新获取三页，然后得到一个列表。

`async def` 内出现 `yield` 时定义的是**异步生成器函数**，调用结果是异步生成器对象，消费方式是 async for；不能把它当成普通协程直接 `await pages()`。

这里的异步推导式也按页逐次等待，**并没有同时发起三页获取**。async for 描述迭代协议，不负责自动创建并发任务。协议定义见 [Python 数据模型：Asynchronous Iterators](https://docs.python.org/3.11/reference/datamodel.html#asynchronous-iterators)。

### 7.2 return、yield、await 再一起比较一次

| 语法 | 控制流含义 | 本篇对应例子 |
| --- | --- | --- |
| `return value` | 本次协程执行结束，交回最终结果 | prepare 返回资料名字 |
| `await operation` | 取得可等待操作的结果，必要时挂起 | 等待资料准备完成 |
| `yield value` | 产出一项，保存继续执行的位置 | 分页生成器产出一页 |

异步生成器中可以用不带值的 `return` 结束，但不能用 `return value` 返回最终值。`yield from` 也不是异步生成器委托语法；转交另一个异步迭代器的数据时，用 async for 逐项 yield。

这些关键字都涉及暂停或结束，但“向消费者产出数据”和“等待另一个操作完成”不是同一件事。分清之后，就不容易把异步生成器、协程和 Task 混成一种对象。

### 7.3 async with 解决“获取和释放资源也可能需要等待”

普通 with 管理进入和离开时的资源处理，例如打开、关闭文件。async with 对应异步的进入与退出过程，底层方法是 `__aenter__()` 和 `__aexit__()`。

先用独立小实验看见资源生命周期。完整脚本 `async_resource.py`：

```python
import asyncio
from contextlib import asynccontextmanager


@asynccontextmanager
async def study_session():
    print("开始连接")
    await asyncio.sleep(0.1)
    resource = {"connected": True}
    print("连接完成")
    try:
        yield resource
    finally:
        print("开始释放")
        await asyncio.sleep(0.1)
        resource["connected"] = False
        print("释放完成")


async def main():
    try:
        async with study_session() as session:
            print("使用资源：", session["connected"])
            raise ValueError("模拟使用阶段失败")
    except ValueError:
        print("外层接到业务错误")
    print("退出后：", session["connected"])


if __name__ == "__main__":
    asyncio.run(main())
```

顺序为连接、使用、释放、外层接到错误，最后连接状态 False。中间出现异常，仍会执行 finally 中的退出工作，然后异常继续传给外层。

`@asynccontextmanager` 把“yield 前准备、yield 处交给调用方、finally 里清理”的结构转换成可用于 async with 的上下文管理器。这里 yield 恰好一次，和上一节不断产出多页的异步生成器职责不同。见 [contextlib.asynccontextmanager](https://docs.python.org/3.11/library/contextlib.html#contextlib.asynccontextmanager)。

async with 不会自动把同步函数变快，也不会自动并发执行它内部的语句。它的价值在于明确资源的拥有范围，并允许进入和退出阶段进行异步等待。

## 8. 任务结果与异常：先决定一项失败后应该怎样

### 8.1 按输入顺序收集，还是先完成的先处理？

gather 适合“等这一组成功完成后，按输入顺序拿到结果”。如果希望先到一份就处理一份，可以使用 `as_completed()`。

完整脚本 `completion_order.py`：

```python
import asyncio


async def prepare(name, delay):
    await asyncio.sleep(delay)
    return name, f"{name}已就绪"


async def main():
    tasks = [
        asyncio.create_task(prepare("A", 0.3)),
        asyncio.create_task(prepare("B", 0.1)),
        asyncio.create_task(prepare("C", 0.2)),
    ]
    for next_result in asyncio.as_completed(tasks):
        name, result = await next_result
        print("先收到：", name, result)


if __name__ == "__main__":
    asyncio.run(main())
```

通常依次收到 B、C、A。循环里虽然一次只 await 一个“下一个结果”，但全部 Task 已提前创建；它没有把工作重新变成串行。

为兼容 Python 3.11，这里采用普通 for 加 await。这个普通迭代接口产出的包装协程不能直接当作原始 Task 的身份标识，所以让工作返回自己的名字，能稳定关联结果与输入。新版本支持的其他迭代方式应按对应版本文档理解。

### 8.2 单个任务失败时，异常会在等待结果处重新出现

完整脚本 `await_error.py`：

```python
import asyncio


async def prepare():
    await asyncio.sleep(0.05)
    raise ValueError("资料内容无法解析")


async def main():
    task = asyncio.create_task(prepare())
    try:
        await task
    except ValueError as error:
        print("调用方处理：", error)
    print("执行结束：", task.done())
    print("属于取消：", task.cancelled())


if __name__ == "__main__":
    asyncio.run(main())
```

最后输出 True、False：任务已经结束，但并非被取消，而是执行失败。能拿到 Task 不代表能拿到正常返回值，等待它时必须遵循业务需要处理异常。

只创建任务，从不等待也不检查异常，可能出现 `Task exception was never retrieved`。这说明任务失败没有被所属流程接住，排查时应找任务的创建者与结果接收者。见 [asyncio 开发指南](https://docs.python.org/3.11/library/asyncio-dev.html#detect-never-retrieved-exceptions)。

### 8.3 独立工作允许部分失败：把成功与错误逐项收集

完整脚本 `collect_results.py`：

```python
import asyncio


async def prepare(index):
    await asyncio.sleep(0.05)
    if index == 2:
        raise ValueError("第 2 项内容错误")
    return f"资料 {index}"


async def main():
    inputs = [1, 2, 3]
    results = await asyncio.gather(
        *(prepare(index) for index in inputs),
        return_exceptions=True,
    )
    for index, result in zip(inputs, results):
        if isinstance(result, BaseException):
            print(index, "失败", type(result).__name__)
        else:
            print(index, "成功", result)


if __name__ == "__main__":
    asyncio.run(main())
```

第 1、3 项成功，第 2 项失败；结果依旧按输入顺序对齐。`return_exceptions=True` 把普通任务失败作为结果元素交回来，调用方不能再把每项都无条件当成字符串。

这里用 `BaseException` 识别结果中的异常对象，是为了也识别取消结果；不是写一个捕获所有退出信号的宽泛 except。外层 gather 自己被取消时，仍会影响未完成的子工作，并不因此变成永远返回一个正常列表。

### 8.4 gather 默认失败行为，不等于取消所有兄弟任务

默认 `gather()` 会把先出现的异常传播给等待者，但不会仅因为一个子任务普通失败就自动取消其他任务。

完整脚本 `gather_failure.py` 用显式等待收尾，避免把事件循环退出混进实验：

```python
import asyncio


async def broken():
    await asyncio.sleep(0.05)
    raise ValueError("A 失败")


async def slow():
    await asyncio.sleep(0.15)
    print("B 正常完成")
    return "B 的结果"


async def main():
    first = asyncio.create_task(broken())
    second = asyncio.create_task(slow())
    try:
        await asyncio.gather(first, second)
    except ValueError:
        print("已经收到 A 的错误")
    print("继续等 B：", await second)


if __name__ == "__main__":
    asyncio.run(main())
```

先看到 A 的错误，之后 B 正常完成。如果错误没有被捕获，导致 main 直接退出，`asyncio.run()` 的结束清理又可能取消残余任务。这是入口清理行为，不能误认为是 gather 本身的失败策略。

这些收集与异常语义见 [Coroutines and Tasks：Running Tasks Concurrently](https://docs.python.org/3.11/library/asyncio-task.html#running-tasks-concurrently)。

### 8.5 一组相关工作共同收尾：TaskGroup

当同一组工作共同组成一次操作，某项失败后其他结果也失去意义，可以用 TaskGroup 管理范围。

完整脚本 `taskgroup_failure.py`：

```python
import asyncio


async def broken():
    await asyncio.sleep(0.05)
    raise ValueError("清单解析失败")


async def slow():
    try:
        await asyncio.sleep(5)
        print("慢任务正常完成")
    finally:
        print("慢任务清理资源")


async def main():
    try:
        async with asyncio.TaskGroup() as group:
            group.create_task(broken())
            group.create_task(slow())
    except* ValueError as errors:
        print("处理本例 ValueError 数量：", len(errors.exceptions))


if __name__ == "__main__":
    asyncio.run(main())
```

约 0.05 秒后，出现慢任务清理与异常数量 1，不应看到慢任务正常完成。普通子任务失败会触发其他任务取消，组等待它们完成清理，然后向外报告异常组。

这里的 async with 是上节的同一种协议：进入组后登记任务，退出组时负责等待和收尾。顺利完成时，离开块以后可以从保存的 Task 取 `.result()`；块内创建完就立刻取结果，则可能尚未完成。

TaskGroup 不会撤销已发生的外部副作用。第一个任务已经写成功的文件或提交的远程操作，不会因为第二个任务失败而自动回滚。

### 8.6 ExceptionGroup 与 except* 为什么需要专门的语法？

普通异常表达一次错误，但并发子任务可能在接近的时间各自失败，或者一个任务失败后，另一个任务清理时又失败。异常组能把多个异常作为一个结构传递。

`except* ValueError` 处理组内匹配 ValueError 的部分，其余未处理类型继续传播。一个异常组可以只有一个异常，不要求必须凑齐两个；TaskGroup 也不保证所有原本可能失败的任务都会运行到失败点，因为有的会提前被取消。

上例中的 `len(errors.exceptions)` 适用于这个只有一层的演示；嵌套异常组可能还有子组，不能总把顶层长度当成全部叶子错误数量。同一个 try 中不能混写普通 except 和 except*。完整语法见 [Python 语言参考：except*](https://docs.python.org/3.11/reference/compound_stmts.html#except-star)。

如果希望所有独立任务都运行完，再一次处理多个失败，可以先用 gather 收集，然后主动构建异常组。这个行为与 TaskGroup 的“某项失败后取消其他项”不同。完整脚本 `collected_exception_group.py`：

```python
import asyncio


async def read_source(index):
    await asyncio.sleep(0.02 * index)
    if index == 1:
        raise ValueError("第一份资料格式错误")
    raise ConnectionError("第二个来源连接失败")


async def main():
    results = await asyncio.gather(
        read_source(1), read_source(2), return_exceptions=True
    )
    errors = [item for item in results if isinstance(item, Exception)]
    if errors:
        raise ExceptionGroup("本批资料失败", errors)


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except* ValueError as errors:
        print("需要修复格式：", len(errors.exceptions))
    except* ConnectionError as errors:
        print("需要检查连接：", len(errors.exceptions))
```

两个处理分支都执行，各报告 1 项。这个实验没有被取消的子任务；因此只提取普通 Exception 来构建 ExceptionGroup。取消继承自 BaseException，不能随意塞进只接受普通 Exception 的组里，也不应默默把取消当成一般业务错误丢弃。实际收集结果时要另行定义取消的传播策略。

### 8.7 回到队列：现在补上失败时的共同收尾

在 `queue_workers.py` 中，只替换 main，保留 producer、consumer 和 STOP：

```python
async def main():
    queue = asyncio.Queue(maxsize=2)
    results = []
    worker_count = 2
    async with asyncio.TaskGroup() as group:
        for index in range(worker_count):
            group.create_task(consumer(f"worker-{index}", queue, results))
        await producer(queue, worker_count)
        await queue.join()
    print("结果：", sorted(results))
```

正常结果不变。现在若某消费者出现未捕获的普通异常，组会取消其余消费者，并中断组体中正在等待的生产或 join 流程，等待清理后抛出异常组，不会把失败工作当成正常完成的一批结果。

练习时在 consumer 的处理分支中临时加入 `if item == 3: raise ValueError("模拟损坏资料")`，观察程序能否结束并报告错误，而不是一直等下去。若业务允许单项失败后继续，应在消费者内部只捕获预期业务异常，记录失败后继续下一项；不要无差别吞掉程序错误。

## 9. 超时与取消：不只是“等不到就算了”

### 9.1 取消是一个请求，还需要任务配合收尾

完整脚本 `cancel_task.py`：

```python
import asyncio


async def worker(started):
    try:
        print("任务开始")
        started.set()
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("任务收到取消")
        raise
    finally:
        print("任务执行清理")


async def main():
    started = asyncio.Event()
    task = asyncio.create_task(worker(started))
    await started.wait()
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("调用方确认取消完成")
    print("已取消：", task.cancelled())


if __name__ == "__main__":
    asyncio.run(main())
```

这里 Event 只用来确保任务确实已经开始，再发取消请求，避免依赖猜测的短暂 sleep。日志依次说明启动、收到取消、清理、调用方确认，最后 True。

`task.cancel()` 不等于工作已经退出，所以调用方还 await task，等待取消完成。`CancelledError` 是取消机制使用的异常，直接继承 `BaseException`，普通 `except Exception` 不会捕获它。

若需要捕获取消做日志或清理，通常应重新抛出；否则任务可能把自己伪装成成功，也可能影响依赖取消实现的 TaskGroup 和 timeout。资源清理可以放在 finally，既覆盖正常退出，也覆盖异常和取消路径。

### 9.2 给一段流程设超时

完整脚本 `timeout_demo.py`：

```python
import asyncio


async def slow_job():
    try:
        await asyncio.sleep(10)
        return "完成"
    finally:
        print("清理慢任务")


async def main():
    try:
        async with asyncio.timeout(0.1):
            await slow_job()
    except TimeoutError:
        print("这一段流程超时")


if __name__ == "__main__":
    asyncio.run(main())
```

约 0.1 秒后先清理，再报告超时。`asyncio.timeout()` 在内部使用取消机制，并在退出上下文时把自己引发的取消转换为 TimeoutError，因此这里在 **async with 外面**捕获它。

如果同一个 timeout 块内先后等待两个操作，预算覆盖整个块，不是每个 await 都重新得到 0.1 秒。若需要每项独立超时，则把 timeout 放到各项工作内部。预算放置位置应与业务含义一致。

### 9.3 wait_for、shield 以及“超时后工作还在不在”

等待单个操作也可以写 `await asyncio.wait_for(operation(), timeout=1)`。超时通常会取消被等待操作，并等待其取消清理，因此实际耗时可能超过名义上的一秒。

`asyncio.shield(task)` 可在特定场景阻止调用者的取消传递给被保护任务，但外层调用者仍会收到取消。它不是一键修复取消错误的工具：用了 shield，仍要保存任务引用，并明确谁负责最终等待和清理。

尤其要注意，事件循环被同步阻塞时，超时处理本身也可能得不到执行机会。它不是一个能在任意 Python 行立刻打断代码的硬性外部计时器。取消请求也不保证远端 HTTP 服务停止了已经开始的业务操作。相关机制见 [Coroutines and Tasks：Timeouts](https://docs.python.org/3.11/library/asyncio-task.html#timeouts)。

## 10. 三个容易让异步代码失效的陷阱

### 10.1 阻塞函数放进 async def，仍然会挡住整个事件循环

完整脚本 `blocking_demo.py`：

```python
import asyncio
import time


def legacy_read():
    time.sleep(0.3)
    return "旧函数的结果"


async def ticker():
    for index in range(4):
        print("心跳", index)
        await asyncio.sleep(0.1)


async def blocking_job():
    print("直接调用开始")
    result = legacy_read()
    print("直接调用结束：", result)


async def threaded_job():
    print("交给线程开始")
    result = await asyncio.to_thread(legacy_read)
    print("交给线程结束：", result)


async def main():
    print("第一轮：观察心跳是否停住")
    await asyncio.gather(ticker(), blocking_job())
    print("第二轮：观察等待时是否还有心跳")
    await asyncio.gather(ticker(), threaded_job())


if __name__ == "__main__":
    asyncio.run(main())
```

第一轮中，直接调用开始与结束之间不会出现心跳，因为事件循环所在的线程被 legacy_read 阻塞。第二轮把同步函数放到工作线程执行，事件循环仍能在等待期间安排 ticker。

`asyncio.to_thread()` 适合将已有阻塞 I/O 函数接入异步流程。注意传入的是函数 `legacy_read`，不是先调用后的 `legacy_read()`，否则阻塞早已在事件循环线程发生。

线程不是无限资源。大量纯 Python CPU 计算也不应仅靠 to_thread 期待多核加速；更重要的是，取消 await to_thread 的任务，通常不能强行终止已经在线程里开始运行的同步函数。需要让旧函数本身有超时或停止机制。阻塞问题见 [asyncio 开发指南：Running Blocking Code](https://docs.python.org/3.11/library/asyncio-dev.html#running-blocking-code)。

### 10.2 单线程仍然会有竞态：问题出在读写之间的等待

完整脚本 `shared_counter.py`：

```python
import asyncio


async def run_round(use_lock):
    count = 0
    lock = asyncio.Lock()

    async def increment():
        nonlocal count
        if use_lock:
            async with lock:
                previous = count
                await asyncio.sleep(0)
                count = previous + 1
        else:
            previous = count
            await asyncio.sleep(0)
            count = previous + 1

    await asyncio.gather(*(increment() for _ in range(20)))
    return count


async def main():
    print("未加锁：", await run_round(False))
    print("加锁：", await run_round(True))


if __name__ == "__main__":
    asyncio.run(main())
```

按本例默认调度，未加锁是 1，加锁是 20。`sleep(0)` 特意让出执行机会：多个任务先读到相同的旧值 0，恢复后分别写入 1，因此更新被覆盖。

锁保证同一时间只有一个任务进入这段读、等、写的整体区域，其他任务必须等它退出后再读新值。这里把 await 放在锁中是为了演示保护跨等待的操作；实际若只是一个不含 await 的本地计数递增，通常没有必要这样制造等待或加锁。

锁的代价是让受保护区域按顺序执行，应尽量缩小范围。asyncio 的锁用于同一异步运行环境中的任务协调，不是跨线程锁，也不是跨进程锁。见 [Synchronization Primitives：Lock](https://docs.python.org/3.11/library/asyncio-sync.html#lock)。

### 10.3 并发多不等于无限快：用 Semaphore 限制进行中的操作

完整脚本 `limited_concurrency.py`：

```python
import asyncio


async def main():
    semaphore = asyncio.Semaphore(2)
    active = 0
    peak = 0

    async def prepare(index):
        nonlocal active, peak
        async with semaphore:
            active += 1
            peak = max(peak, active)
            try:
                print("开始", index, "进行中", active)
                await asyncio.sleep(0.1)
                return index
            finally:
                active -= 1

    results = await asyncio.gather(*(prepare(index) for index in range(5)))
    print("结果：", results)
    print("最大同时处理数：", peak)


if __name__ == "__main__":
    asyncio.run(main())
```

结果是 `[0, 1, 2, 3, 4]`，峰值为 2。进入 async with 时申请名额，离开时归还；异常或取消离开也会执行退出逻辑。

Semaphore 限制的是某段代码的同时进入数量，不是每秒请求数。请求极快时，同时只有 2 个，也可能一秒完成很多次。每秒限额需要单独的速率控制策略。

它也不限制任务创建数量：本例仍会为全部 5 项安排工作，只是部分任务等在入口外。几十万项工作要控制排队规模时，更适合固定消费者加有界 Queue。锁用于独占，信号量允许有限多个同时进入，有界队列管理等待处理的工作存量；这三个工具解决的问题各不相同。见 [Semaphore](https://docs.python.org/3.11/library/asyncio-sync.html#semaphore)。

## 11. 从模拟等待走向真实 HTTP 请求

### 11.1 替换等待操作，不需要推翻前面的任务组织

此前 prepare 中的 sleep 只是为了看清调度。真实网络请求还涉及连接、状态码、响应体、超时和资源释放，但外层仍然是“安排多个独立工作 → 等待 → 收集结果”。

在学习环境安装：

```bash
python -m pip install "aiohttp>=3.9,<4"
```

使用 aiohttp 是因为它提供配合 asyncio 的 HTTP 客户端。不要把同步 `requests.get()` 放在协程里再加一个 await，期待它因此变成异步。

先看一条请求。完整脚本 `one_request.py`：

```python
import asyncio
import aiohttp


async def main():
    timeout = aiohttp.ClientTimeout(total=10)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with session.get("https://example.com") as response:
            response.raise_for_status()
            text = await response.text()
            print("HTTP 状态：", response.status)
            print("正文字符数：", len(text))


if __name__ == "__main__":
    asyncio.run(main())
```

成功时打印状态码和实际正文长度；公网结果受网络和网站行为影响，不能要求固定字节数。`response.status` 是已经拿到的响应元信息，读取完整正文则可能仍要等待更多数据，所以 `response.text()` 要 await。

`raise_for_status()` 把 HTTP 错误状态转换为异常；“收到响应”不等于“得到成功业务结果”。即使是 HTTP 200，也还需要检查返回内容是否符合应用预期。

### 11.2 两层 async with 分别管理什么？

外层 ClientSession 管理会话、连接池等资源。同一批请求通常共享一个 Session，使连接可以复用；不应为每个 URL 都无条件新建 Session。

内层管理一次响应的生命周期，包括正文读取和离开时的资源释放。若状态检查或读正文失败，正常的异常退出仍会走上下文清理。Session 应在异步运行期间创建，并在所有使用它的任务结束后关闭。

一次 `await response.text()` 会读入完整响应体。对于很大的文件或无限流，应使用分块读取和逐步处理，不能因为是异步 I/O 就忽略内存占用。基础客户端用法见 [aiohttp Client Quickstart](https://docs.aiohttp.org/en/stable/client_quickstart.html)。

### 11.3 一批请求：共享会话、限制并发、逐项记录预期错误

完整脚本 `http_checks.py`：

```python
import asyncio
import aiohttp


async def check(session, semaphore, url):
    try:
        async with semaphore:
            async with session.get(url) as response:
                response.raise_for_status()
                text = await response.text()
                return {
                    "url": url,
                    "ok": True,
                    "status": response.status,
                    "characters": len(text),
                }
    except (aiohttp.ClientError, TimeoutError) as error:
        return {
            "url": url,
            "ok": False,
            "error": type(error).__name__,
        }


async def main():
    urls = ["https://example.com", "https://www.python.org"]
    semaphore = asyncio.Semaphore(2)
    timeout = aiohttp.ClientTimeout(total=10)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with asyncio.TaskGroup() as group:
            tasks = [
                group.create_task(check(session, semaphore, url))
                for url in urls
            ]
        results = [task.result() for task in tasks]
    for result in results:
        print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

从外往里读：Session 的范围包住 TaskGroup，组的范围包住所有请求；组退出时任务已经收尾，然后才关闭 Session。结果按 tasks 列表的输入顺序提取，任务完成顺序可以不同。

对可预期的网络和超时错误，check 返回失败记录，其他独立请求继续。若内部写错变量名等未知程序错误没有被捕获，TaskGroup 会把它作为异常处理，而不会悄悄当成“网站访问失败”。本例没有捕获 CancelledError，因此上层取消仍能向下传递。

`ClientTimeout(total=10)` 从 aiohttp 请求过程开始计算，不包含进入 semaphore 之前排队等待名额的时间。如果业务定义是“从提交任务算起总共最多十秒”，可以把 `asyncio.timeout(10)` 放在工作函数中、semaphore 外侧。相同的数字放在不同位置，实际限制的时间段不同。

超时参数还可以区分连接和读取阶段；连接池也有自己的数量上限。初次学习先明确一个整体请求超时和一个任务并发上限，遇到具体瓶颈时再拆细。见 [aiohttp Client Reference：ClientTimeout](https://docs.aiohttp.org/en/stable/client_reference.html#aiohttp.ClientTimeout)。

### 11.4 重试不是在 except 里无限循环

有些失败可能是暂时的，例如连接中断；有些不是，例如错误 URL、权限失败或响应数据持续不符合约定。重试前要明确错误类型、尝试次数、间隔和总时间预算。

等待重试间隔用 `await asyncio.sleep(...)`，不要用 `time.sleep(...)` 阻塞循环。对于有副作用的请求，还要知道第一次是否可能已经在远端成功；本地超时只说明本地没及时取得结果。不能盲目重发一个可能重复创建订单的操作。

本篇 GET 检查没有自动重试，便于先看清一次请求的结果与资源生命周期。独立练习时可以为临时连接错误加入有限次数重试，并分别记录“尝试失败”和“最终失败”。

## 12. 综合实验：并发获取、逐项校验、区分三种失败

### 12.1 先定义结果分类，再写并发代码

我们要获取四份课程记录，其中一份合法、一份字段错误、一份连接失败、一份等待过久。网络返回成功只说明拿到了数据，还需要上一篇的 Pydantic 检查字段。

这次使用可控的模拟来源，不依赖公网，使四种结果每次都能稳定复现。代码里的等待代表外部 I/O，数据验证仍然是同步的 CPU 工作：小记录可以直接验证，大批复杂计算则需要测量是否会阻塞事件循环。

完整脚本 `validated_fetch.py`。只依赖 Pydantic 与标准库，不需要 aiohttp，也不需要导入上一篇的脚本：

```python
import asyncio
from pydantic import BaseModel, ConfigDict, Field, ValidationError


class Course(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)
    title: str = Field(min_length=1)
    lessons: int = Field(gt=0)


async def fetch_raw(index):
    if index == 4:
        await asyncio.sleep(10)
    else:
        await asyncio.sleep(0.02)
    if index == 3:
        raise ConnectionError("模拟无法连接数据源")
    if index == 2:
        return {"title": "   ", "lessons": 0}
    return {"title": "  asyncio 入门  ", "lessons": "12"}


async def load_one(index, semaphore):
    try:
        async with asyncio.timeout(0.2):
            async with semaphore:
                raw = await fetch_raw(index)
    except TimeoutError:
        return {"id": index, "ok": False, "stage": "timeout"}
    except ConnectionError:
        return {"id": index, "ok": False, "stage": "network"}

    try:
        course = Course.model_validate(raw)
    except ValidationError as error:
        return {
            "id": index,
            "ok": False,
            "stage": "validation",
            "fields": [list(item["loc"]) for item in error.errors()],
        }

    return {"id": index, "ok": True, "data": course.model_dump(mode="json")}


async def main():
    semaphore = asyncio.Semaphore(2)
    async with asyncio.TaskGroup() as group:
        tasks = [group.create_task(load_one(i, semaphore)) for i in range(1, 5)]
    for task in tasks:
        print(task.result())


if __name__ == "__main__":
    asyncio.run(main())
```

预期结果：

```text
{'id': 1, 'ok': True, 'data': {'title': 'asyncio 入门', 'lessons': 12}}
{'id': 2, 'ok': False, 'stage': 'validation', 'fields': [['title'], ['lessons']]}
{'id': 3, 'ok': False, 'stage': 'network'}
{'id': 4, 'ok': False, 'stage': 'timeout'}
```

### 12.2 沿着一条成功路径读，再沿着失败路径读

第 1 项等待并发名额，模拟取得字典后，退出 semaphore 归还名额。Pydantic 去掉标题空白，将课时字符串解析为整数，然后导出 JSON 兼容字典。

第 2 项取得数据没有失败，但标题与课时不合规则，所以 stage 是 validation。第 3 项根本没有拿到原始记录，属于 network。第 4 项在给定时间内没有完成获取，属于 timeout。把错误分开，才能决定是提示上游修数据、检查连接，还是调整等待策略。

超时放在 semaphore 外，所以从 load_one 开始等待时就计算预算，包括排队。对于前 3 项，这个示例留有足够时间完成短等待；第 4 项长等待触发取消。退出 semaphore 时会归还名额，预期超时被转换成失败记录。

验证发生在网络超时块之后，因此这个 0.2 秒预算并不包括后续同步校验。即使把同步校验写进 timeout，也不能保证循环被长计算阻塞时立刻超时；要解决的是执行模型本身。

### 12.3 为什么使用 TaskGroup，却仍然允许部分失败？

因为预期失败已经在 load_one 内转成了结构化结果，TaskGroup 看到的是正常返回。若出现没有被设计为普通业务失败的异常，组仍会取消相关任务并向上传播。

这是一种明确策略：可以预计并展示给用户的失败逐项收集，未预期的程序错误不要隐藏。若业务要求任一资料失败就整组失败，调整的是错误处理策略，而不是仅把 gather 改成 TaskGroup 的名字。

下一步接入真实 HTTP 时，替换 fetch_raw 的实现，并像第 11 节一样传入共享 Session、检查状态、读取 JSON；模型验证的位置和失败分类仍然保留。不要用“HTTP 返回了 200”替代业务数据验证。

## 13. 自测与进一步实践

### 13.1 先预测，不打开解释

1. 连续写 `await prepare("A")`、`await prepare("B")`，为什么可能耗时相加？
2. 先 create_task 两次，再先后 await 两个 Task，为什么可以并发？
3. await 一个立即返回的协程，是否一定让其他任务运行？
4. `queue.empty()` 为 True，是否说明消费者都处理完了？
5. 两个消费者只收到一个 STOP，会发生什么？
6. 使用 `gather(..., return_exceptions=True)` 后，结果是不是一定全是业务数据？
7. TaskGroup 某项普通失败后，其他任务的清理在哪里发生？已发生的外部操作是否撤销？
8. `asyncio.timeout(1)` 能否保证一个同步阻塞函数在一秒内被强行停止？
9. Semaphore(2) 是否意味着每秒最多两次请求？是否意味着只创建两个 Task？
10. `async for` 是否自动并发获取所有页？

<details markdown="1">
<summary>展开参考答案</summary>

1. B 在 A 返回后才创建和开始；事件循环不能提前运行尚未安排的工作。
2. 两个任务都提前安排，等待 first 的同时 second 也能推进。
3. 不一定。只有实际挂起才提供该次调度切换机会。
4. 不说明。取出的项目可能仍在处理；join 等待的是未完成计数归零。
5. 一个退出，另一个可能永远等待下一项。结束信号应覆盖所有消费者。
6. 不是，结果可能包含异常对象，需要逐项区分成功与失败。
7. 组请求取消其余任务，任务在 finally 或上下文退出中清理；组等待收尾再报告异常。外部副作用不会自动回滚。
8. 不能。阻塞会妨碍事件循环处理超时，线程中的同步工作也不会自动被强制停止。
9. 两者都不是。信号量限制同时进入受保护区域的数量，不是频率或全部任务数。
10. 不会。它按异步迭代协议逐项等待，并发必须另外组织。

</details>

### 13.2 三个逐步增加难度的作业

**第一步：从空文件重做等待实验。** 准备 A、B、C 三项，等待时间设为 0.3、0.1、0.2 秒。分别实现串行、gather、as_completed。记录总耗时、完成顺序、结果列表顺序，并解释三者为何不同。

**第二步：把一批工作改成有限消费者。** 生产 20 项数据、设置 3 名消费者、队列容量 4。记录处理中的最大数量和队列长度；给某项加入预期业务失败，并选择记录失败后继续。最后再加入一个未预期异常，观察 TaskGroup 是否能让整个程序结束。

**第三步：把校验案例接入真实数据。** 在自己控制的本地服务中准备合法 JSON、字段错误 JSON、错误状态码和慢响应四个入口。先逐个请求，再并发请求，检查每项错误是否落到正确阶段。真实请求先在本地验证，能避免把公网波动误认为程序逻辑问题。

<details markdown="1">
<summary>完成后用这些条件验收</summary>

第一步：串行约 0.6 秒，并发约 0.3 秒；gather 结果仍按 A、B、C 排列，as_completed 通常先收到 B、C、A。不要把非常接近的完成时间顺序写成业务依赖。

第二步：最多 3 项由消费者同时处理；最多 4 项留在队列里，二者分别计数。每次成功 get 都对应一次 task_done，结束信号覆盖所有消费者。预期失败有记录，未知错误不会导致 join 永久等待。

第三步：连接错误、HTTP 错误、JSON 解析错误、模型验证错误和超时应根据需要分别记录；Session 的生命周期覆盖使用它的全部任务；结果带回输入标识；测试结束后没有未等待协程、未收取异常或未关闭资源警告。

</details>

### 13.3 写完代码以后，按现象回查

| 现象 | 优先检查 |
| --- | --- |
| 总时间仍接近各项之和 | 是否在创建下一项之前就 await 了上一项 |
| 所有任务一起卡住 | 是否调用了同步阻塞库或长 CPU 循环 |
| 创建了函数调用却没日志 | 是否只创建了协程对象，没有驱动它 |
| 主流程结束，后台工作没完成 | 谁负责保存和等待这些 Task |
| 队列程序一直退出不了 | 消费者是否存活，结束信号数量、task_done 配对是否正确 |
| 一项失败后的行为与预期不同 | 区分 gather、TaskGroup、工作函数内部异常处理与入口清理 |
| 超时后仍有外部活动 | 检查线程任务、远端副作用、shield 与真正的取消边界 |
| 网络成功但后续计算失败 | 检查 JSON 解析与模型验证是否缺失 |

这些问题可以用调试模式继续观察，例如脚本入口使用 `asyncio.run(main(), debug=True)`。先建立一条任务从创建到收尾的完整路径，再增加任务数量，日志才不会变成无法解释的交错文本。
