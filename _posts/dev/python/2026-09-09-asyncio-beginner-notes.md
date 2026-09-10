---
title: "Python 的 asyncio：一次动手实践之旅（Real Python 译文）"
author: gpt6_astra
date: 2026-09-09 08:10:00 +0800
last_modified_at: 2026-09-10
categories: [Dev, Python]
tags: [python, asyncio, translation, realpython]
description: "Real Python 教程《Python's asyncio: A Hands-On Walkthrough》的中文翻译，涵盖协程与 async/await、事件循环、协程链与队列、异步迭代器与 async with、任务调度与异常组，以及异步生态常用库。"
toc: true
---

> **译文说明**：本文是 Real Python 教程 [Python's asyncio: A Hands-On Walkthrough](https://realpython.com/async-io-python/)（作者 Leodanis Pozo Ramos）的中文翻译，仅供个人学习使用，版权归原作者与 Real Python 所有。原文中的广告、推荐课程、测验推广与订阅引导已删除；代码块按原文示例重新排版，并修正了影响理解或示例结果的个别错误；并发关系图沿用原文配图。
>
> **运行说明**：示例已在 Python 3.13.13 下核对，网站请求使用 aiohttp 3.14.3。耗时、时钟、对象地址、随机任务日志和网站状态码均为示例，不保证每次相同。标有 `>>>` 的代码使用普通 Python REPL，只有“asyncio REPL”一节使用支持顶层 `await` 的交互环境。

Python 的 `asyncio` 库让你可以用 `async` 和 `await` 关键字编写并发代码。Python 异步 I/O 的核心构件是可等待对象（awaitable object）——最常见的是协程（coroutine）——由事件循环（event loop）负责调度并异步执行。这套编程模型让你能在单个执行线程内高效地管理多个 I/O 密集型（I/O-bound）任务。

在本教程中，你将学习 Python `asyncio` 的工作方式、如何定义并运行协程，以及在对执行 I/O 密集型任务的应用中，何时该用异步编程来换取更好的性能。

**学完本教程，你将理解以下几点：**

* Python 的 **`asyncio`** 提供了一个框架，用**协程**、**事件循环**与**非阻塞 I/O 操作**来编写**单线程并发代码**。
* 对 I/O 密集型任务而言，异步 I/O **往往能胜过多线程（multithreading）**——尤其在需要管理大量并发任务时——因为它省去了线程管理的开销。
* 当你的应用把大量时间花在等待 **I/O 操作**上（例如网络请求或文件访问），而你又希望**在不额外创建线程或进程的前提下并发运行许多这类任务**时，就应该使用 `asyncio`。

通过一系列动手示例，你将获得用 `asyncio` 编写高效 Python 代码的实用技能，让程序在 I/O 需求不断增长时依然能优雅地扩展。

## 初步认识异步 I/O（A First Look at Async I/O）

在深入 `asyncio` 之前，值得先花点时间把异步 I/O 与其他并发模型做个对比，看看它在 Python 那幅广阔、有时甚至令人眼花缭乱的图景中处于什么位置。先来看几个基础概念：

* **并行（parallelism）** 指同时执行多个操作。
* **多进程（multiprocessing）** 是一种实现并行的手段，它把任务分散到计算机的各个 CPU 核心上。多进程非常适合 CPU 密集型（CPU-bound）任务，例如紧密的 [`for` 循环](https://realpython.com/python-for-loop/)和数学计算。
* **并发（concurrency）** 是一个比并行稍宽泛的说法，指多个任务具备以重叠方式运行的能力。并发并不必然意味着并行。
* **线程（threading）** 是一种并发执行模型，多个线程轮流执行任务。一个进程可以包含多个线程。由于[全局解释器锁（GIL）](https://realpython.com/python-gil/)，Python 与线程的关系有些复杂，不过这超出了本教程的范围。

线程适合处理[**I/O 密集型任务**](https://realpython.com/ref/glossary/io-bound-task/)。I/O 密集型任务的绝大部分时间都耗在等待[**输入/输出（I/O）**](https://realpython.com/ref/glossary/input-output/)完成上；而 [CPU 密集型任务](https://realpython.com/ref/glossary/cpu-bound-task/)的特征则是 CPU 核心从开始到结束一直满负荷运转。

Python [标准库](https://realpython.com/ref/glossary/standard-library/)长期以来通过 `multiprocessing`、`concurrent.futures` 和 `threading` 包[支持上述这些模型](https://docs.python.org/3/library/concurrency.html)。

现在该往这个组合里加入一位新成员了。近年来，另一种模型被更完整地构建进了 [CPython](https://realpython.com/cpython-source-code-guide/)：**异步 I/O**，通常简称 **async I/O**。该模型由标准库中的 [**`asyncio`**](https://realpython.com/ref/stdlib/asyncio/) 包以及 [`async`](https://realpython.com/python-keywords/#the-async-keyword) 与 [`await`](https://realpython.com/python-keywords/#the-await-keyword) 两个关键字共同提供。

**注意：** 异步 I/O 并不是新概念。它在 [Go](https://gobyexample.com/goroutines)、[C#](https://docs.microsoft.com/en-us/dotnet/csharp/async) 和 [Rust](https://doc.rust-lang.org/book/ch17-00-async-await.html) 等语言中已经存在，或者正在被加入这些语言。

Python 文档把 `asyncio` 包介绍为一个[编写并发代码的库](https://docs.python.org/3/library/asyncio.html)。不过，异步 I/O 既不是线程也不是多进程，它并不建立在这两者之上。

异步 I/O 是一种单线程、单进程的技术，采用[协作式多任务（cooperative multitasking）](https://en.wikipedia.org/wiki/Cooperative_multitasking)。它在单进程单线程的前提下，让多个任务的执行过程交错推进。[协程](https://realpython.com/ref/glossary/coroutine/)——简称 **coro**——是异步 I/O 的核心特性，可以被并发地调度，但协程本身并不天然具备并发性。

再强调一次：异步 I/O 是一种并发编程模型，但它不是并行。它与线程的关系比与多进程更近，但又与两者都不同，是并发生态中一个独立存在的成员。

还剩一个术语没解释：说某个东西是**异步的（asynchronous）**，到底是什么意思？这里给不出严格定义，但就本教程而言，你可以把握两个关键性质：

1. **异步例程**可以在等待结果的过程中*暂停*自己的执行，让其他例程趁机运行。
2. **异步代码**通过协调各个异步例程，来促成任务的并发执行。

下面这张示意图把上述内容串了起来。白色术语代表概念，绿色术语代表这些概念的具体实现方式：

![并发包含并行：线程与异步 I/O 可以实现并发，多进程可以实现并行](/assets/img/posts/python/concurrency-parallelism-realpython.png){: width="504" height="411" }
_并发与并行的关系。图片来源：[Real Python 原文](https://realpython.com/async-io-python/)。_

图中的内圈表示并行，外圈表示更广义的并发：并行是并发的一种情形，但并发不一定是并行。本教程中的单线程 `asyncio` 可以并发推进多个任务，但不会同时执行这些任务的 Python 代码。

如果你想彻底弄清线程、多进程与异步 I/O 之间的区别，不妨先停下来看看[《Speed Up Your Python Program With Concurrency》](https://realpython.com/python-concurrency/)这篇教程。眼下我们先聚焦异步 I/O。

### 异步 I/O 到底怎么回事（Async I/O Explained）

异步 I/O 乍看之下似乎反直觉、甚至自相矛盾：一个用于支撑并发代码的东西，怎么可能只用单个 CPU 核心上的单个线程？Miguel Grinberg 在 [PyCon](https://realpython.com/pycon-guide/) 上的演讲把这件事讲得非常漂亮：

> 国际象棋大师朱迪特·波尔加（Judit Polgár）举办一场车轮战表演，她要同时与多位业余棋手对弈。她有两种组织表演的方式：*同步*方式与*异步*方式。
>
> 前提假设：
>
> * 24 位对手
> * 朱迪特每走一步棋用 5 秒
> * 每位对手每走一步用 55 秒
> * 每盘棋平均 30 个回合（双方合计 60 步）
>
> **同步版本**：朱迪特一盘一盘地下，绝不同时下两盘，直到一盘结束。每盘棋耗时 *(55 + 5) \* 30 == 1800* 秒，也就是 30 分钟。整场表演耗时 *24 \* 30 == 720* 分钟，即 **12 小时**。
>
> **异步版本**：朱迪特在各张棋桌之间走动，每到一桌走一步。她离开棋桌，让对手在等待期间思考下一步。在全部 24 盘棋上各走一步，朱迪特需要 *24 \* 5 == 120* 秒，也就是 2 分钟。整场表演因此缩短到 *120 \* 30 == 3600* 秒，仅仅 **1 小时**。（[来源](https://youtu.be/iG6fr81xHKA?t=4m29s)）

世界上只有一个朱迪特·波尔加，她一次只能走一步棋。而以异步方式表演，却把时间从 12 小时压缩到 1 小时。异步 I/O 就是把这一原理搬到了编程中：在异步 I/O 里，程序的事件循环——后面还会详细讲——运行着多个任务，让每个任务都能在最佳的时机轮流推进。

异步 I/O 会接管那些耗时很长的[函数](https://realpython.com/defining-your-own-python-function/)——就像上面例子中的一整盘棋局——它们原本会阻塞程序的执行（也就是朱迪特的时间），并以某种方式管理它们，好让其他函数能在这段空档里运行。在棋类的例子里，朱迪特就是在对手思考落子的时候去和其他参与者对弈。

### 异步 I/O 并不简单（Async I/O Isn't Simple）

编写经得起考验的多线程代码可能相当困难，而且容易出错。异步 I/O 回避了你在多线程设计中可能遇到的一些绊脚石。但这并不意味着[异步编程](https://realpython.com/ref/glossary/asynchronous-programming/)在 Python 里就是件轻松的事。

要意识到，一旦你稍微深入表层之下，异步编程就会变得棘手。Python 的异步模型建立在回调（callback）、协程、事件（event）、传输（transport）、协议（protocol）以及[未来对象（future）](https://docs.python.org/3/library/asyncio-future.html#asyncio.Future)等概念之上——光是这些术语就足以让人望而生畏。

话虽如此，Python 异步编程的生态已经大有改善。`asyncio` 包已经成熟，如今提供了一套稳定的 [API](https://realpython.com/ref/glossary/api/)。此外，它的文档也经过了大幅重写，关于这个主题还涌现出了一些高质量的参考资料。

## 用 asyncio 编写 Python 异步 I/O（Async I/O in Python With `asyncio`）

既然你已经对异步 I/O 这一并发模型有了一些背景认识，接下来就该探索 Python 的具体实现了。Python 的 `asyncio` 包与它相关的两个关键字 [`async`](https://realpython.com/python-keywords/#the-async-keyword) 和 [`await`](https://realpython.com/python-keywords/#the-await-keyword) 各有分工，但合在一起就能帮你声明、构建、执行并管理异步代码。

### 协程与协程函数（Coroutines and Coroutine Functions）

异步 I/O 的核心是[**协程**](https://realpython.com/ref/glossary/coroutine/)这一概念：它是一种可以暂停执行、稍后再恢复的对象。在暂停期间，它可以把控制权交给事件循环，由事件循环去执行另一个协程。协程对象由调用[**协程函数**](https://realpython.com/ref/glossary/coroutine-function/)（也叫**异步函数**）产生，而协程函数用 `async def` 结构来定义。

在写下第一段异步代码之前，先看一个同步运行的例子：

文件名：`countsync.py`

```python
import time


def count():
    print("One")
    time.sleep(1)
    print("Two")
    time.sleep(1)


def main():
    for _ in range(3):
        count()


if __name__ == "__main__":
    start = time.perf_counter()
    main()
    elapsed = time.perf_counter() - start
    print(f"{__file__} executed in {elapsed:0.2f} seconds.")
```

`count()` 函数先[打印](https://realpython.com/python-print/) `One` 并等待一秒，再打印 `Two` 并再等一秒。[`main()`](https://realpython.com/python-main-function/) 函数里的循环会执行 `count()` 三次。而在 [`if __name__ == "__main__"`](https://realpython.com/if-name-main-python/) 条件块中，你在执行之初记录当前时间，调用 `main()`，算出总耗时并显示在屏幕上。

[运行这个脚本](https://realpython.com/run-python-scripts/)，你会得到如下输出：

```shell
$ python countsync.py
One
Two
One
Two
One
Two
countsync.py executed in 6.03 seconds.
```

脚本交替打印 `One` 和 `Two`，每次打印之间间隔一秒。总共耗时略多于六秒。

如果把这个脚本改成使用 Python 的异步 I/O 模型，大致会是下面这样：

文件名：`countasync.py`

```python
import asyncio


async def count():
    print("One")
    await asyncio.sleep(1)
    print("Two")
    await asyncio.sleep(1)


async def main():
    await asyncio.gather(count(), count(), count())


if __name__ == "__main__":
    import time
    start = time.perf_counter()
    asyncio.run(main())
    elapsed = time.perf_counter() - start
    print(f"{__file__} executed in {elapsed:0.2f} seconds.")
```

这里你用 `async` 关键字把 `count()` 变成了协程函数：它打印 `One`，等待一秒，再打印 `Two`，再等待一秒。你用 `await` 关键字来*等待* `asyncio.sleep()` 的执行。这会把控制权交还给程序的事件循环，相当于在说：*我要睡一秒，你趁这段时间去跑别的东西吧。*

`main()` 是另一个协程函数，它用 [`asyncio.gather()`](#其他-asyncio-工具other-asyncio-tools) 并发运行三个 `count()` 实例。你用 `asyncio.run()` 函数来启动[事件循环](#异步-io-事件循环the-async-io-event-loop)并执行 `main()`。

把这个版本与同步版本的性能做个对比：

```shell
$ python countasync.py
One
One
One
Two
Two
Two
countasync.py executed in 2.00 seconds.
```

得益于异步 I/O 的思路，总执行时间从六秒多降到两秒出头，这正体现了 `asyncio` 处理 I/O 密集型任务的效率。

想弄清异步版本*为什么*结束得更早，可以把两个版本放在同一条时间轴上，看看时间究竟花在哪里：

```text
同步版本（countsync.py，总计约 6 秒）
时间(秒)  0    1    2    3    4    5    6
count#1   ├─One─┤─Two─┤
count#2                  ├─One─┤─Two─┤
count#3                                   ├─One─┤─Two─┤
          └── 三次调用首尾相接，等待时间无法重叠 ──┘

异步版本（countasync.py，总计约 2 秒）
时间(秒)  0    1    2
count#1   ├─One─┤─Two─┤
count#2   ├─One─┤─Two─┤
count#3   ├─One─┤─Two─┤
          └── 三个协程的等待同时进行，只有睡眠时间被真正等待 ──┘
```

以上为示意框图，用于还原原文中交互式并发时间线图所表达的含义：三个协程都在同一时刻开始等待，因此总耗时约等于一条协程链中的两次一秒睡眠，也就是约两秒，而不是三条链依次执行所需的约六秒。

虽然 `time.sleep()` 和 `asyncio.sleep()` 看起来平平无奇，但它们在这里是耗时过程的替身，都涉及等待时间。对 `time.sleep()` 的调用可以代表一次耗时的阻塞式函数调用，而 `asyncio.sleep()` 则用来代表一次同样需要时间才能完成的[非阻塞调用](https://realpython.com/ref/glossary/non-blocking-operation/)。

正如你将在下一节看到的，等待某个对象（包括 `asyncio.sleep()`）的好处在于：当前函数可以暂时把控制权让给另一个更能立刻做事的函数。相比之下，`time.sleep()` 或任何其他阻塞调用都与异步 Python 代码不兼容，因为它会在整个睡眠期间让一切停滞。

### async 与 await 关键字（The `async` and `await` Keywords）

到这里，该更正式地定义 `async`、`await` 以及它们帮你创建的那些协程函数了：

* **`async def`** 语法结构引入的是一个**协程函数**或一个[**异步生成器（asynchronous generator）**](https://realpython.com/ref/glossary/asynchronous-generator/)。
* **`async with`** 与 **`async for`** 语法结构分别引入异步的 **`with` 语句**与异步的 **`for` 循环**。
* **`await`** 关键字用于等待一个可等待对象。当被等待的操作需要挂起时，当前任务会让出执行权，事件循环便可运行其他就绪任务；如果结果能立即取得，就可能直接继续执行。

为了把最后一点说得更清楚：当 Python 在 `g()` 协程的作用域中遇到 `await f()` 表达式时，`g()` 会等待 `f()` 的结果。如果 `f()` 在等待过程中挂起，事件循环就有机会运行其他任务；如果 `f()` 能立即完成，则不一定发生任务切换。

写成代码，最后一条大致如下：

```python
async def g():
    result = await f()  # 暂停，等 f() 返回后再回到 g()
    return result
```

围绕 `async` 和 `await` 的使用时机与方式，还有一套严格的规则。无论你是在熟悉语法，还是已经接触过 `async` 和 `await`，这些规则都很有帮助：

* 用 `async def` 结构可以定义协程函数。它可以使用 `await`、`return` 或 `yield`，但这些都是可选的：

  * 普通协程函数中可以使用 `await`、`return`，或者两者都用。要调用协程函数，你必须 `await` 它以取得结果，或者直接在事件循环中运行它。
  * 在 `async def` 函数中使用 `yield` 会创建异步生成器。要遍历这个生成器，你可以使用 [`async for` 循环或推导式](#异步迭代器循环与推导式async-iterators-loops-and-comprehensions)。
  * `async def` 中不能使用 `yield from`，否则会抛出 [`SyntaxError`](https://realpython.com/invalid-syntax-python/)。
* 在普通 Python 脚本中，`await` 必须出现在 `async def` 函数体内，否则会抛出 `SyntaxError`。后文的 asyncio REPL 是支持顶层 `await` 的特殊交互环境。

下面这几个简短示例概括了上述规则：

```python
async def f(x):
    y = await z(x)  # 可以——协程中允许 `await` 和 `return`
    return y


async def g(x):
    yield x  # 可以——这是一个异步生成器


async def m(x):
    yield from gen(x)  # 不行——会抛出 SyntaxError


def n(x):
    y = await z(x)  # 不行——会抛出 SyntaxError（这里没有 `async def`）
    return y
```

最后，当你使用 `await f()` 时，要求 `f()` 是一个[**可等待对象（awaitable）**](https://realpython.com/ref/glossary/awaitable/)，也就是另一个协程，或者定义了 `.__await__()` [特殊方法](https://realpython.com/python-magic-methods/)并返回迭代器的对象。绝大多数情况下，你只需要关心协程。

下面是一个更精细的例子，展示异步 I/O 如何压缩等待时间。假设你有一个名为 `makerandom()` 的协程函数，它不断产生 [0, 10] 范围内的随机整数，直到某个数超过阈值就返回。在下面的例子里，你把这个函数异步地运行三次。为了区分每次调用，你用不同颜色来标记：

文件名：`rand.py`

```python
import asyncio
import random

COLORS = (
    "\033[0m",   # 颜色结束
    "\033[36m",  # 青色
    "\033[91m",  # 红色
    "\033[35m",  # 品红
)


async def main():
    return await asyncio.gather(
        makerandom(1, 9),
        makerandom(2, 8),
        makerandom(3, 8),
    )


async def makerandom(delay, threshold=6):
    color = COLORS[delay]
    print(f"{color}Initiated makerandom({delay}).")
    while (number := random.randint(0, 10)) <= threshold:
        print(f"{color}makerandom({delay}) == {number} too low; retrying.")
        await asyncio.sleep(delay)
    print(f"{color}---> Finished: makerandom({delay}) == {number} " + COLORS[0])
    return number


if __name__ == "__main__":
    random.seed(444)
    r1, r2, r3 = asyncio.run(main())
    print()
    print(f"r1: {r1}, r2: {r2}, r3: {r3}")
```

这段带颜色的输出胜过千言万语。下面用一张时间线示意还原原文中动画演示的执行过程（终端里每一行会以对应颜色显示）：

```text
时间  →
delay=1（青色）  Initiated makerandom(1). ── sleep 1 ── 重试 ── ... ──> 返回 10（超过阈值 9）
delay=2（红色）  Initiated makerandom(2). ──── sleep 2 ──── 重试 ── ... ──> 返回 9 或 10（超过阈值 8）
delay=3（品红）  Initiated makerandom(3). ────── sleep 3 ────── 重试 ── ... ──> 返回 9 或 10（超过阈值 8）

三个协程交替打印、交替睡眠：某个协程在 asyncio.sleep() 上等待时，
事件循环立刻切去运行另一个已经就绪的协程，于是三份输出交错出现。
```

以上为示意图，展示“三个协程交替推进、日志交错”的效果，不代表精确的完成顺序。本次运行的结果为 `r1: 10, r2: 10, r3: 9`；其中第一个协程的阈值为 9，所以生成 9 时仍会重试，只有 10 才满足返回条件。

这个程序定义了 `makerandom()` 协程，并用三个不同的输入并发运行它。大多数程序都由许多小而模块化的协程，外加一个负责[串联](#协程链coroutine-chaining)它们的包装函数组成。在 `main()` 中，你把这三种任务收集到一起。这三次对 `makerandom()` 的调用就是你的**任务池（pool of tasks）**。

本例中生成随机数的部分是 CPU 密集型任务，但它带来的影响可以忽略不计。`asyncio.sleep()` 模拟的是一个 I/O 密集型任务，也恰好说明了：只有 I/O 密集型或非阻塞的任务，才能从异步 I/O 模型中获益。

### 异步 I/O 事件循环（The Async I/O Event Loop）

在异步编程中，事件循环就像一个[无限循环](https://realpython.com/python-while-loop/#intentional-infinite-loops)：它监视各个协程，收集关于谁处于空闲状态的反馈，并四处寻找这段时间里可以执行的东西。当某个空闲协程所等待的条件变为可用时，它能够把该协程唤醒。

在现代 Python 中，启动事件循环的推荐做法是使用 [`asyncio.run()`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run)。这个函数负责创建事件循环、运行入口协程、清理剩余任务并关闭循环。当同一线程中已经有事件循环在运行时，你不能调用这个函数。

你也可以用 [`get_running_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop) 函数拿到正在运行的事件循环实例：

```python
loop = asyncio.get_running_loop()
```

如果你需要在 Python 程序内部与事件循环交互，上面这种写法是个不错的选择。`loop` 对象支持用 `.is_running()` 和 `.is_closed()` 做自我检查。举例来说，当你想通过把事件循环作为参数传出去来[调度一个回调](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio-example-lowlevel-helloworld)时，这就很有用。注意，如果当前没有正在运行的事件循环，`get_running_loop()` 会抛出 [`RuntimeError`](https://realpython.com/ref/builtin-exceptions/runtimeerror/) 异常。

更重要的是理解事件循环表层之下发生了什么。有几点值得强调：

* 协程在被绑定到事件循环之前，自己几乎做不了什么。
* 默认情况下，异步事件循环运行在单个线程、单个 CPU 核心上。在大多数 `asyncio` 应用中只会有一个事件循环，通常位于主线程。在不同线程中运行多个事件循环在技术上可行，但通常没有必要，也不推荐。
* 事件循环是可插拔的。你可以编写自己的实现，让它像 `asyncio` 内置的事件循环一样运行任务。

关于第一点：如果你有一个等待其他协程的协程，那么单独调用它几乎不会产生任何效果：

```python
>>> import asyncio
>>> async def main():
...     print("Hello...")
...     await asyncio.sleep(1)
...     print("World!")
...
>>> routine = main()
>>> routine
<coroutine object main at 0x0000000000000000>
```

在这个例子中，直接调用 `main()` 返回一个协程对象，你无法单独使用它。你需要用 `asyncio.run()` 把 `main()` 协程调度到事件循环上执行：

```python
>>> asyncio.run(routine)
Hello...
World!
```

通常你会把自己的 `main()` 协程包在 `asyncio.run()` 调用里。而更低层的协程，可以用 `await` 来执行。

最后，事件循环*可插拔*这一点意味着：你可以使用任何一个可用的事件循环实现，这与你的协程结构无关。`asyncio` 包自带两种不同的[事件循环实现](https://docs.python.org/3/library/asyncio-eventloop.html#event-loop-implementations)。

默认使用哪种事件循环实现，取决于你的平台和 Python 版本。例如在 Unix 上默认通常是 [`SelectorEventLoop`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.SelectorEventLoop)，而 Windows 上则使用 [`ProactorEventLoop`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.ProactorEventLoop)，以获得更好的子进程与 I/O 支持。

第三方事件循环实现同样可用。例如 [uvloop](https://github.com/MagicStack/uvloop) 包提供了另一种实现，号称比 `asyncio` 自带的循环更快。

### asyncio REPL（The `asyncio` REPL）

从 [Python 3.8](https://realpython.com/python38-new-features/) 开始，`asyncio` 模块内置了一个专门的交互式 shell，称为 [asyncio REPL](https://docs.python.org/3/library/asyncio.html#asyncio-cli)。这个环境允许你直接在顶层使用 `await`，无需把代码包进 `asyncio.run()` 调用里。它非常适合用来试验、调试和学习 Python 中的 `asyncio`。

要启动这个 [REPL](https://realpython.com/ref/glossary/repl/)，可以运行以下命令：

```shell
$ python -m asyncio
asyncio REPL 3.13.3 (main, Jun 25 2025, 17:27:59) [Clang 17.0.0 (clang-1700.0.13.3)] on darwin
Use "await" directly instead of "asyncio.run()".
Type "help", "copyright", "credits" or "license" for more information.
>>> import asyncio
>>>
```

一旦出现 `>>>` 提示符，你就可以在里面运行异步代码了。看下面的例子，它复用了上一节的代码：

适用于 Python 3.8+：

```python
>>> import asyncio
>>> async def main():
...     print("Hello...")
...     await asyncio.sleep(1)
...     print("World!")
...
>>> await main()
Hello...
World!
```

这个例子的效果与上一节完全相同。区别在于，你不再用 `asyncio.run()` 运行 `main()`，而是直接使用 `await`。

## 常见的异步 I/O 编程模式（Common Async I/O Programming Patterns）

异步 I/O 有一套自己的可用编程模式，能帮你写出更好的异步代码。在实践中，你可以*把协程串联起来*，也可以使用协程的[队列](https://realpython.com/ref/glossary/queue/)。下面几节将介绍这两种模式的用法。

### 协程链（Coroutine Chaining）

协程的一个关键特性是你可以把它们*串联*起来。别忘了，协程是可等待对象，所以另一个协程可以用 `await` 关键字来等待它。这使得把程序拆分成更小、更易管理、可复用的协程变得更容易。

下面的例子模拟了一个获取用户信息的两个步骤的过程：第一步获取用户信息，第二步获取该用户发布的文章：

文件名：`chained.py`

```python
import asyncio
import random
import time


async def main():
    user_ids = [1, 2, 3]
    start = time.perf_counter()
    await asyncio.gather(
        *(get_user_with_posts(user_id) for user_id in user_ids)
    )
    end = time.perf_counter()
    print(f"\n==> Total time: {end - start:.2f} seconds")


async def get_user_with_posts(user_id):
    user = await fetch_user(user_id)
    await fetch_posts(user)


async def fetch_user(user_id):
    delay = random.uniform(0.5, 2.0)
    print(f"User coro: fetching user by {user_id=}...")
    await asyncio.sleep(delay)
    user = {"id": user_id, "name": f"User{user_id}"}
    print(f"User coro: fetched user with {user_id=} (done in {delay:.1f}s).")
    return user


async def fetch_posts(user):
    delay = random.uniform(0.5, 2.0)
    print(f"Post coro: retrieving posts for {user['name']}...")
    await asyncio.sleep(delay)
    posts = [f"Post {i} by {user['name']}" for i in range(1, 3)]
    print(
        f"Post coro: got {len(posts)} posts by {user['name']} "
        f"(done in {delay:.1f}s):"
    )
    for post in posts:
        print(f"- {post}")


if __name__ == "__main__":
    random.seed(444)
    asyncio.run(main())
```

在这个例子中，你定义了 `fetch_user()` 和 `fetch_posts()` 两个主要协程。两者都用 `asyncio.sleep()` 加上随机延迟来模拟一次网络调用。

在 `fetch_user()` 协程中，你返回一个模拟的用户[字典](https://realpython.com/python-dicts/)。在 `fetch_posts()` 中，你用这个字典构造并打印归属于当前用户的模拟文章列表；该函数没有显式的 `return`，所以返回值是 `None`。随机延迟用来模拟网络延迟这类真实世界中的异步行为。

协程链发生在 `get_user_with_posts()` 里。这个协程等待 `fetch_user()`，并把结果存进 `user` [变量](https://realpython.com/python-variables/)。等用户信息到手后，它被传给 `fetch_posts()`，以异步方式取出文章。

在 `main()` 中，你用 `asyncio.gather()` 并发运行这些串联起来的协程：按用户 ID 的数量执行相应次数的 `get_user_with_posts()`。

执行该脚本的结果如下：

```shell
$ python chained.py
User coro: fetching user by user_id=1...
User coro: fetching user by user_id=2...
User coro: fetching user by user_id=3...
User coro: fetched user with user_id=2 (done in 0.5s).
Post coro: retrieving posts for User2...
User coro: fetched user with user_id=1 (done in 1.0s).
Post coro: retrieving posts for User1...
User coro: fetched user with user_id=3 (done in 1.2s).
Post coro: retrieving posts for User3...
Post coro: got 2 posts by User2 (done in 1.8s):
- Post 1 by User2
- Post 2 by User2
Post coro: got 2 posts by User1 (done in 1.6s):
- Post 1 by User1
- Post 2 by User1
Post coro: got 2 posts by User3 (done in 1.5s):
- Post 1 by User3
- Post 2 by User3

==> Total time: 2.68 seconds
```

输出中的 `user_id=1`、`User2`、`Post 1 by User2` 分别来自 f-string 表达式 `{user_id=}`、`{user['name']}` 与 `{post}` 的插值结果。

如果把所有操作的耗时加总，用同步实现大约需要 7.6 秒；而改用异步实现后，只用了 2.68 秒。

这种“等待一个协程、再把结果传给下一个”的模式构成了**协程链（coroutine chain）**，其中每一步都依赖前一步。这个例子模拟了一种常见的异步工作流：先拿到一份信息，再用它去获取相关联的数据。

### 协程与队列的结合（Coroutine and Queue Integration）

`asyncio` 包提供了若干[类队列的类](https://realpython.com/queue-in-python/#using-asynchronous-queues)，它们的设计与 [`queue`](https://docs.python.org/3/library/queue.html#module-queue) 模块中的[类](https://realpython.com/python-classes/)相似。在前面几个例子中，你还不需要队列结构。在 `chained.py` 里，每个任务由一个协程完成，你再把协程串联起来，让数据一步步传递下去。

另一种做法是使用往[队列](https://realpython.com/ref/glossary/queue/)中添加条目的**生产者（producer）**。每个生产者可以在错开的、随机的、事先无法预知的时刻往队列里放入多个条目。然后，一组**消费者（consumer）**在条目出现时把它们取走，贪婪地处理，不需要等待任何其他信号。

在这种设计中，生产者与消费者之间没有链式依赖。消费者不知道生产者有多少个，反之亦然。

单个生产者或消费者往队列中添加、取出条目所需的时间长短不一。队列充当了一个吞吐通道，让双方无需直接对话就能彼此通信。

下面是 `chained.py` 基于队列的改写版本：

文件名：`queued.py`

```python
import asyncio
import random
import time


async def main():
    queue = asyncio.Queue()
    user_ids = [1, 2, 3]
    start = time.perf_counter()
    await asyncio.gather(
        producer(queue, user_ids),
        *(consumer(queue) for _ in user_ids),
    )
    end = time.perf_counter()
    print(f"\n==> Total time: {end - start:.2f} seconds")


async def producer(queue, user_ids):
    async def fetch_user(user_id):
        delay = random.uniform(0.5, 2.0)
        print(f"Producer: fetching user by {user_id=}...")
        await asyncio.sleep(delay)
        user = {"id": user_id, "name": f"User{user_id}"}
        print(f"Producer: fetched user with {user_id=} (done in {delay:.1f}s)")
        await queue.put(user)

    await asyncio.gather(*(fetch_user(uid) for uid in user_ids))
    for _ in range(len(user_ids)):
        await queue.put(None)  # 哨兵值，用于通知消费者结束


async def consumer(queue):
    while True:
        user = await queue.get()
        if user is None:
            break
        delay = random.uniform(0.5, 2.0)
        print(f"Consumer: retrieving posts for {user['name']}...")
        await asyncio.sleep(delay)
        posts = [f"Post {i} by {user['name']}" for i in range(1, 3)]
        print(
            f"Consumer: got {len(posts)} posts by {user['name']} "
            f"(done in {delay:.1f}s):"
        )
        for post in posts:
            print(f"- {post}")


if __name__ == "__main__":
    random.seed(444)
    asyncio.run(main())
```

在这个例子中，`producer()` 函数异步地获取模拟的用户数据。每个取到的用户字典都被放进一个 `asyncio.Queue` 对象，由它把数据共享给消费者。在把所有用户对象都生产出来之后，生产者插入一个[哨兵值（sentinel value）](https://en.wikipedia.org/wiki/Sentinel_value)——在这个语境下也叫[毒丸（poison pill）](https://realpython.com/queue-in-python/#killing-a-worker-with-the-poison-pill)——给每个消费者，用来表示不会再有数据送来，好让消费者干净地退出。

`consumer()` 函数持续从队列中读取。如果读到的是用户字典，它就模拟获取该用户的文章，等待一段随机延迟，然后打印结果；如果读到的是哨兵值，就跳出循环并结束。

这种解耦让多个消费者可以并发处理用户，即便生产者还在继续生成用户；队列则保证了生产者与消费者之间安全、有序的通信。

队列是生产者与消费者之间的通信点，使整个系统具备可扩展性与良好的响应能力。

这段代码的实际运行结果如下：

```shell
$ python queued.py
Producer: fetching user by user_id=1...
Producer: fetching user by user_id=2...
Producer: fetching user by user_id=3...
Producer: fetched user with user_id=2 (done in 0.5s)
Consumer: retrieving posts for User2...
Producer: fetched user with user_id=1 (done in 1.0s)
Consumer: retrieving posts for User1...
Producer: fetched user with user_id=3 (done in 1.2s)
Consumer: retrieving posts for User3...
Consumer: got 2 posts by User2 (done in 1.8s):
- Post 1 by User2
- Post 2 by User2
Consumer: got 2 posts by User1 (done in 1.6s):
- Post 1 by User1
- Post 2 by User1
Consumer: got 2 posts by User3 (done in 1.5s):
- Post 1 by User3
- Post 2 by User3

==> Total time: 2.68 seconds
```

同样，这段代码只用了 2.68 秒，比同步方案更高效。结果与上一节使用协程链时基本一致。

## Python 中其他的异步 I/O 特性（Other Async I/O Features in Python）

Python 的异步 I/O 特性并不止 `async def` 和 `await` 这两个结构。它还包含其他高级工具，让异步编程更具表达力，也与常规 Python 结构更一致。

下面几节将探索一些强大的异步特性，包括异步循环与推导式、`async with` 语句以及异常组。它们能帮你写出更干净、更易读的异步代码。

### 异步迭代器、循环与推导式（Async Iterators, Loops, and Comprehensions）

除了用 `async` 和 `await` 创建协程之外，Python 还提供了 `async for` 结构，用于遍历一个[**异步迭代器（asynchronous iterator）**](https://realpython.com/ref/glossary/asynchronous-iterator/)。异步迭代器让你可以遍历异步生成的数据。循环运行期间，它会把控制权交还给事件循环，好让其他异步任务得以运行。

**注意：** 想进一步了解异步迭代器，可以看看[《Asynchronous Iterators and Iterables in Python》](https://realpython.com/python-async-iterators/)这篇教程。

这个概念的一个自然延伸是[**异步生成器**](https://realpython.com/ref/glossary/asynchronous-generator/)。下面这个例子生成 2 的幂，并在循环与推导式中使用它们：

```python
>>> import asyncio
>>> async def powers_of_two(stop=10):
...     exponent = 0
...     while exponent < stop:
...         yield 2**exponent
...         exponent += 1
...         await asyncio.sleep(0.2)  # 模拟一些异步工作
...
>>> async def main():
...     g = []
...     async for i in powers_of_two(5):
...         g.append(i)
...     print(g)
...     f = [j async for j in powers_of_two(5) if not (j // 3 % 5)]
...     print(f)
...
>>> asyncio.run(main())
[1, 2, 4, 8, 16]
[1, 2, 16]
```

同步生成器、循环、推导式与它们的异步版本之间有一个关键区别：异步版本并不会天然让迭代变成并发。它们只是允许你在显式使用 `await` 让出控制权时，让事件循环在两次迭代之间去运行其他任务。迭代本身仍然是顺序的；如果还需要并发处理取得的数据，可以另外调度任务，例如使用 `asyncio.gather()`。

对于只实现异步迭代协议或异步上下文管理协议的对象，应分别使用 `async for` 或 `async with`；普通的 `for` 或 `with` 无法处理这类对象。

### 异步 with 语句（Async `with` Statements）

[`with` 语句](https://realpython.com/python-with-statement/)也有它的[异步](https://realpython.com/ref/glossary/asynchronous-programming/)版本：`async with`。这个结构在异步代码中相当常见，因为许多 [I/O 密集型任务](https://realpython.com/ref/glossary/io-bound-task/)都包含准备与收尾阶段。

举个例子：假设你需要写一个协程来检查某些网站是否在线。为此，你可以使用 [`aiohttp`](https://docs.aiohttp.org/en/stable/index.html)，这是一个第三方库，需要在命令行运行 `python -m pip install aiohttp` 来安装。

下面是一个实现该功能的简短示例：

```python
>>> import asyncio
>>> import aiohttp
>>> async def check(url):
...     async with aiohttp.ClientSession() as session:
...         async with session.get(url) as response:
...             print(f"{url}: status -> {response.status}")
...
>>> async def main():
...     websites = [
...         "https://realpython.com",
...         "https://pycoders.com",
...         "https://www.python.org",
...     ]
...     await asyncio.gather(*(check(url) for url in websites))
...
>>> asyncio.run(main())
https://www.python.org: status -> 200
https://pycoders.com: status -> 200
https://realpython.com: status -> 200
```

在这个例子中，你用 `aiohttp` 和 `asyncio` 对一组网站并发执行 [HTTP GET](https://realpython.com/api-integration-in-python/#get) 请求。`check()` 协程获取并打印网站的状态码。`async with` 语句确保 `ClientSession` 和每个 HTTP 响应都被正确、异步地管理：它们的开启与关闭都不会阻塞事件循环。

在这个例子中，使用 `async with` 保证了底层的网络资源——包括连接与套接字——即使在发生错误时也能被正确释放。

最后，`main()` 并发运行各个 `check()` 协程，让你可以并行获取这些 URL，而不必等一个结束再开始下一个。

### 其他 asyncio 工具（Other `asyncio` Tools）

除了 `asyncio.run()`，你还用过其他几个包级函数，例如 `asyncio.gather()` 和 `asyncio.get_running_loop()`。你可以用 [`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) 在运行中的事件循环里调度协程对象。下面在 `main()` 中创建任务，再通过 `asyncio.run(main())` 运行整个程序：

```python
>>> import asyncio
>>> async def coro(numbers):
...     await asyncio.sleep(min(numbers))
...     return list(reversed(numbers))
...
>>> async def main():
...     task = asyncio.create_task(coro([3, 2, 1]))
...     print(f"{type(task) = }")
...     print(f"{task.done() = }")
...     return await task
...
>>> result = asyncio.run(main())
type(task) = <class '_asyncio.Task'>
task.done() = False
>>> print(f"result: {result}")
result: [1, 2, 3]
```

这个模式里有一个你需要留意的微妙细节：如果你用 `create_task()` 创建了任务，而 `main()` 返回时它们还没完成，`asyncio.run()` 会在关闭事件循环前取消这些剩余任务。为了确保任务完成，通常应显式等待它们，或使用 `gather()`、`TaskGroup` 等方式管理它们的生命周期。

`create_task()` 函数把一个协程对象包装成更高级的 [`Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) 对象，并把它调度到事件循环上在后台并发运行。相比之下，直接等待一个协程会立刻运行它，并暂停调用方的执行，直到被等待的协程结束。

`gather()` 函数则用于把一组协程整齐地放进一个**未来对象（future object）**。该对象代表一个起初未知、但终将在某个时刻可用的结果占位符，通常就是异步计算的结果。

如果你等待 `gather()`，并传入多个任务或协程，那么在所有任务正常完成时，你会得到它们的全部结果。`gather()` 的结果将是一个列表，其中按输入顺序给出各个结果：

```python
>>> import time
>>> async def main():
...     task1 = asyncio.create_task(coro([10, 5, 2]))
...     task2 = asyncio.create_task(coro([3, 2, 1]))
...     print("Start:", time.strftime("%X"))
...     result = await asyncio.gather(task1, task2)
...     print("End:", time.strftime("%X"))
...     print(f"Both tasks done: {all((task1.done(), task2.done()))}")
...     return result
...
>>> result = asyncio.run(main())
Start: 14:38:49
End: 14:38:51
Both tasks done: True
>>> print(f"result: {result}")
result: [[2, 5, 10], [1, 2, 3]]
```

在正常完成的情况下，`gather()` 会等待这一组协程全部出结果，返回列表的顺序与输入顺序一致。需要注意，默认 `return_exceptions=False` 时，第一个异常会立即向等待方传播，其他任务不会因此自动取消；后文将演示如何用 `return_exceptions=True` 收集各个任务的异常。

另一种做法是遍历 `asyncio.as_completed()`，按完成顺序获取结果。像下面这样使用普通 `for` 时，每次取得的是一个协程对象，等待它就能得到下一个已完成任务的结果；这个对象不是原始的 `Task`。Python 3.13 起也支持用 `async for` 遍历，传入 Task 或 Future 时可取得原对象。在下面这段代码中，`coro([3, 2, 1])` 的结果会在 `coro([10, 5, 2])` 完成之前就可取到，而在使用 `gather()` 函数时并非如此：

```python
>>> async def main():
...     task1 = asyncio.create_task(coro([10, 5, 2]))
...     task2 = asyncio.create_task(coro([3, 2, 1]))
...     print("Start:", time.strftime("%X"))
...     for task in asyncio.as_completed([task1, task2]):
...         result = await task
...         print(f"result: {result} completed at {time.strftime('%X')}")
...     print("End:", time.strftime("%X"))
...     print(f"Both tasks done: {all((task1.done(), task2.done()))}")
...
>>> asyncio.run(main())
Start: 14:36:36
result: [1, 2, 3] completed at 14:36:37
result: [2, 5, 10] completed at 14:36:38
End: 14:36:38
Both tasks done: True
```

在这个例子中，`main()` 用 `asyncio.as_completed()` 取得一系列可等待的协程，并通过 `await` 按任务完成顺序获取结果，因此不必先等较慢的任务完成。

结果是：较快的那个任务（`task2`，等待 1 秒）先结束，它的结果也更早打印；而耗时更长的任务（`task1`，等待 2 秒）随后完成并打印。当你需要按任务完成的节奏动态处理它们时，`as_completed()` 很有用，这能提升并发工作流的响应速度。

### 异步异常处理（Async Exception Handling）

从 [Python 3.11](https://realpython.com/python311-new-features/) 开始，你可以用 [`ExceptionGroup`](https://realpython.com/python311-exception-groups/) 类来处理可能并发发生的多个互不相关的异常。当多个协程各自可能抛出不同异常时，这一点尤其有用。此外，新的 `except*` 语法能帮你优雅地同时处理多个错误。

下面快速演示在异步代码中如何使用这个类：

适用于 Python 3.11+：

```python
>>> import asyncio
>>> async def coro_a():
...     await asyncio.sleep(1)
...     raise ValueError("Error in coro A")
...
>>> async def coro_b():
...     await asyncio.sleep(2)
...     raise TypeError("Error in coro B")
...
>>> async def coro_c():
...     await asyncio.sleep(0.5)
...     raise IndexError("Error in coro C")
...
>>> async def main():
...     results = await asyncio.gather(
...         coro_a(),
...         coro_b(),
...         coro_c(),
...         return_exceptions=True
...     )
...     exceptions = [e for e in results if isinstance(e, Exception)]
...     if exceptions:
...         raise ExceptionGroup("Errors", exceptions)
...
```

在这个例子中，你有三个协程，分别抛出三种不同类型的[异常](https://realpython.com/python-built-in-exceptions/)。在 `main()` 函数里，你把这几个协程作为参数调用 `gather()`。同时你把 `return_exceptions` 参数设为 `True`，以便在异常发生时把它们捕获下来。

接着，你用列表推导式把这些异常存入一个新列表。如果该列表至少包含一个异常，你就为它们创建一个 `ExceptionGroup`。

要处理这个异常组，可以使用下面的代码：

适用于 Python 3.11+：

```python
>>> try:
...     asyncio.run(main())
... except* ValueError as ve_group:
...     print(f"[ValueError handled] {ve_group.exceptions}")
... except* TypeError as te_group:
...     print(f"[TypeError handled] {te_group.exceptions}")
... except* IndexError as ie_group:
...     print(f"[IndexError handled] {ie_group.exceptions}")
...
[ValueError handled] (ValueError('Error in coro A'),)
[TypeError handled] (TypeError('Error in coro B'),)
[IndexError handled] (IndexError('Error in coro C'),)
```

在这段代码中，你把对 `asyncio.run()` 的调用包在一个 [`try`](https://realpython.com/ref/keywords/try/) 块里。然后，你用 `except*` 语法分别捕获预期的异常。在每一个分支中，你都往屏幕上打印一条错误信息。

## 把异步 I/O 放进实际语境（Async I/O in Context）

现在你已经看过了足够多的异步代码，不妨退一步想想：什么时候异步 I/O 才是理想选择？又该如何判断它是否合适，或者是否该换用其他并发模型？

### 何时该用异步 I/O（When to Use Async I/O）

对执行阻塞操作的函数——例如标准文件 I/O 或同步网络请求——使用 `async def`，会阻塞整个事件循环，抵消异步 I/O 的好处，还很可能降低程序的效率。只对[非阻塞操作](https://realpython.com/ref/glossary/non-blocking-operation/)使用 `async def` 函数。

异步 I/O 与多进程之间并不是一场真正的对决。如果你愿意，完全可以[把两种模型结合起来用](https://youtu.be/0kXaLh8Fz3k?t=10m30s)。在实践中，如果你有多个 CPU 密集型任务，多进程才是正确的选择。

异步 I/O 与线程之间的较量则更直接。线程并不简单，即便在某些看起来容易实现线程的场景里，它仍可能因为[竞态条件（race condition）](https://realpython.com/python-thread-lock/#race-conditions)和内存占用等问题，带来难以追踪的 bug。

另外，线程的扩展性通常不如异步 I/O，因为线程是一种数量有限的系统资源。在很多机器上创建上千个线程会直接失败，或者拖慢你的代码。相比之下，创建上千个异步 I/O 任务完全可行。

当你有多个 I/O 密集型任务、其时间主要耗在阻塞式等待上时，异步 I/O 就能大放异彩，例如：

* **网络 I/O**，无论你的程序扮演服务端还是客户端
* **多用户通信应用**，例如群聊或点对点网络，其中存在大量需要并发等待的网络操作
* **多个独立的读/写操作**，使用支持异步调用的库，让它们的等待时间重叠；如果访问共享状态，仍需考虑同步与任务生命周期

不用异步 I/O 的最大理由是：`await` 只支持一组特定的对象，这些对象需要定义一组特定的方法。举例来说，如果你想对某种[数据库管理系统（DBMS）](https://en.wikipedia.org/wiki/Database#Database_management_system)做异步读取，就需要找到该 DBMS 支持 `async` 和 `await` 语法的 Python 封装库。

### 支持异步 I/O 的库（Libraries Supporting Async I/O）

在 Python 中，你会找到不少高质量、支持或构建于 `asyncio` 之上的第三方库与框架，涵盖 Web 服务器、数据库、网络、测试等方向。以下是一些最值得关注的：

* **Web 框架：**
  * [FastAPI](https://fastapi.tiangolo.com/)：用于构建 [Web API](https://realpython.com/python-api/) 的现代异步 Web 框架。
  * [Starlette](https://www.starlette.io/)：轻量级的[异步服务器网关接口（ASGI）](https://en.wikipedia.org/wiki/Asynchronous_Server_Gateway_Interface)框架，用于构建高性能异步 Web 应用。
  * [Sanic](https://sanic.dev/)：为追求速度而构建、基于 `asyncio` 的异步 Web 框架。
  * [Quart](https://github.com/pallets/quart)：异步 Web 微框架，API 与 [Flask](https://realpython.com/flask-project/) 相同。
  * [Tornado](https://github.com/tornadoweb/tornado)：高性能 Web 框架与异步网络库。
* **ASGI 服务器：**
  * [uvicorn](https://www.uvicorn.org/)：快速的 ASGI Web 服务器。
  * [Hypercorn](https://pypi.org/project/Hypercorn/)：支持多种协议与配置选项的 ASGI 服务器。
* **网络工具：**
  * [aiohttp](https://docs.aiohttp.org/)：基于 `asyncio` 的 HTTP 客户端与服务器实现。
  * [HTTPX](https://www.python-httpx.org/)：功能完备的异步与同步 HTTP 客户端。
  * [websockets](https://websockets.readthedocs.io/)：用 `asyncio` 构建 WebSocket 服务器与客户端的库。
  * [aiosmtplib](https://aiosmtplib.readthedocs.io/)：用于[发送邮件](https://realpython.com/python-send-email/)的异步 SMTP 客户端。
* **数据库工具：**
  * [Databases](https://www.encode.io/databases/)：与 [SQLAlchemy](https://realpython.com/python-sqlite-sqlalchemy/) 核心兼容的异步数据库访问层。
  * [Tortoise ORM](https://tortoise.github.io/)：轻量级异步对象关系映射器（ORM）。
  * [Gino](https://python-gino.org/)：构建于 SQLAlchemy 核心之上、面向 [PostgreSQL](https://realpython.com/python-sql-libraries/#postgresql) 的异步 ORM。
  * [Motor](https://motor.readthedocs.io/)：原文列出的异步 [MongoDB](https://realpython.com/introduction-to-mongodb-and-python/) 驱动。**版本补充（2026-09）**：官方已宣布自 2026-05-14 起弃用 Motor，并推荐迁移到 PyMongo Async。
* **实用工具库：**
  * [aiofiles](https://github.com/Tinche/aiofiles)：包装 Python 文件 API，使其可用于 `async` 和 `await`。
  * [aiocache](https://github.com/aio-libs/aiocache)：支持 [Redis](https://realpython.com/python-redis/) 与 Memcached 的异步缓存库。
  * [APScheduler](https://github.com/agronholm/apscheduler)：支持异步作业的任务调度器。
  * [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)：为使用 [pytest](https://realpython.com/pytest-python-testing/) 测试异步函数提供支持。

这些库与框架能帮你写出高性能的异步 Python 应用。无论你是在构建 Web 服务器、从网络获取数据，还是访问数据库，像这样的 `asyncio` 工具都能让你以极小的开销并发处理大量任务。

## 结语（Conclusion）

你已经对 Python 的 `asyncio` 库以及 `async`、`await` 语法有了扎实的理解，也学到了异步编程如何让多个 I/O 密集型任务在单个线程中得到高效管理。

一路走来，你探索了并发、并行、线程、多进程与异步 I/O 之间的区别，也动手实践了基于协程、事件循环、协程链和队列并发的示例。此外，你还了解了 `asyncio` 的高级特性，包括异步上下文管理器、异步迭代器、推导式，以及如何借助第三方异步库。

在构建可扩展的网络服务器、Web API，或需要执行大量同时发生的 I/O 密集型操作的应用时，掌握 `asyncio` 是必不可少的。

**在本教程中，你学会了如何：**

* **区分**各种并发模型，并判断何时该对 I/O 密集型任务使用 **`asyncio`**
* 使用 `async def` 与 `await` **编写、运行并串联协程**
* **管理事件循环**，并用 `asyncio.run()`、`gather()` 和 `create_task()` 调度多个任务
* 实现**协程链**与**异步队列**这类异步模式，用于生产者–消费者工作流
* **使用 `async for`、`async with` 等高级异步特性**，并与**第三方异步库**集成

有了这些技能，你就可以着手构建高性能的现代 Python 应用，让它们能够异步处理大量操作。

## 常见问题（Frequently Asked Questions）

现在你已经对 Python 中的 `asyncio` 有了一些实践经验，可以用下面的问答来检验自己的理解，并回顾所学内容。

这些常见问题都围绕本教程中最重要的概念。原文中点击每个问题旁的 *Show/Hide* 开关即可显示答案，下面直接给出对应的问答内容。

**问：什么是 `asyncio`，它有什么用？**

你用 `asyncio` 配合 `async` 和 `await` 关键字来编写并发代码，从而在单个线程中高效管理多个 I/O 密集型任务，而不会阻塞程序。

**问：对 I/O 密集型工作，为什么 `asyncio` 通常比线程性能更好？**

对于 I/O 密集型工作，`asyncio` 通常能带来更好的性能，因为它避免了线程带来的开销与复杂性。它主要通过重叠 I/O 等待并减少线程管理开销来提高并发效率；这不意味着绕过 GIL，也不会让单线程中的 CPU 密集型 Python 代码并行执行。

**问：什么时候该用 `asyncio`？**

当你的程序把大量时间花在等待 I/O 密集型操作上——例如网络请求或文件访问——而你又希望并发、高效地运行许多这类任务时，就该使用 `asyncio`。

**问：如何定义并运行一个协程？**

你用 `async def` 语法定义协程。要运行它，要么把它传给 `asyncio.run()`，要么用 `asyncio.create_task()` 把它调度为一个任务。

**问：事件循环负责做什么？**

你依靠事件循环来管理协程的调度与执行，让每个协程在它等待某个 I/O 密集型操作或该操作完成时都有机会运行。
