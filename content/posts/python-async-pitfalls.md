---
title: "你的 asyncio 代码可能在假装异步——Python async/await 避坑指南"
date: 2026-07-04T22:00:00+08:00
tags: ["python", "async", "asyncio", "concurrency", "coroutine", "debugging"]
author: "MaRuiZhi"
slug: "python-async-pitfalls"
description: "从真实线上翻车案例出发，深入剖析 Python async/await 的 3 大陷阱、CancelledError 正确处理、TaskGroup 结构化并发，以及测试与实战的最佳实践。"
draft: false
---

# 你的 asyncio 代码可能在假装"异步"——Python async/await 避坑指南

---

你刚把一个同步 Web 服务改成了 `async def`，信心满满地部署上线。压测一跑，P95 延迟从 180ms 飙升到了 1400ms——比改写前还慢了一个数量级。你盯着满屏的 `async` 和 `await`，怀疑人生。

这不是虚构的故事。这是我从无数 Python 团队那里反复看到的真实案例。async/await 看起来简单——加个 `async`、加个 `await`，完事。但它的陷阱比你想象的多得多。

本文不会从"什么是协程"讲起。假设你已经写过几行 async 代码，我们来聊聊那些让你线上翻车的坑，以及怎么爬出来。

---

## 一、先搞清楚你在玩什么游戏

asyncio 最容易被误解的一点：**它不是多线程，更不是并行**。

你只有一个线程、一个 CPU 核心。asyncio 做的是"协作式多任务"——当一个 coroutine 在等 I/O 的时候，另一个可以接上。仅此而已。

```python
import asyncio
import time

async def say_hello():
    print("Hello")
    time.sleep(1)        # 这里阻塞了整个 event loop！
    print("World")

async def say_goodbye():
    await asyncio.sleep(0)  # 即使无事可等，也让出控制权给 event loop
    print("Goodbye")

async def main():
    await asyncio.gather(say_hello(), say_goodbye())

asyncio.run(main())
```

输出是什么？`Hello` → 等 1 秒 → `World` → `Goodbye`。`time.sleep(1)` 阻塞了 event loop，`say_goodbye()` 根本没机会在你"等"的时候插队——它的 `await asyncio.sleep(0)` 要等 event loop 恢复控制后才能执行。

**为什么重要**：你把函数标记成 `async def`，不代表里面的代码就不阻塞了。`async` 只是给你一张"可以挂起的入场券"——你必须真的用 `await` 去挂起，event loop 才能调度别人。

修复方式：

```python
async def say_hello():
    print("Hello")
    await asyncio.sleep(1)   # 挂起，让出控制权
    print("World")
```

输出变成 `Hello` → `Goodbye` →（1 秒后）→ `World`。这才是真正的并发。

---

## 二、踩坑 Top 3：你一定见过至少一个

### 陷阱 1：`RuntimeWarning: coroutine was never awaited`

```python
async def fetch(url):
    return f"data from {url}"

async def main():
    fetch("https://example.com")   # 忘了 await
    print("done")
```

你调了 `fetch()`，返回了一个 coroutine 对象——但你既没 `await` 它，也没用 `create_task()` 调度它。Python 在垃圾回收时发现这个 coroutine 从来没被执行过，就会打印警告。

**为什么重要**：这个警告不是噪声。它意味着你有一段逻辑**永远没有执行**。在测试环境可能看不出来，上了生产就是数据丢失。

修复：要么 `await fetch(url)`，要么 `task = asyncio.create_task(fetch(url))`。

### 陷阱 2：不小心把 CancelledError 吞了

```python
async def worker():
    try:
        await long_running_task()
    except Exception as e:          # Exception 涵盖了 CancelledError（Python 3.8 之前）
        logger.error(f"failed: {e}")
        # 没有 re-raise！
```

在 Python 3.8 之前，`asyncio.CancelledError` 继承自 `Exception`，所以这段代码会静默吞掉取消信号。即使在 3.9+（CancelledError 改继承 BaseException），如果你用了 `except BaseException` 或者手动取消了任务却没 re-raise，任务看起来"结束"了，但调用方并不知道它被取消了——这会导致资源泄漏、状态不一致。

**正确的写法**：

```python
import asyncio

async def worker():
    try:
        await asyncio.sleep(10)     # 模拟长时间任务
    except asyncio.CancelledError:
        # 清理资源
        print("清理中...")
        raise                        # 必须 re-raise
    except Exception as e:
        print(f"unexpected: {e}")

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(0.1)         # 等 worker 启动
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")

asyncio.run(main())
```

输出：
```
清理中...
任务已取消
```

**为什么重要**：CancelledError 是 asyncio 的"刹车信号"。你把刹车踩了然后假装没事发生，车并不会停下来——它只是失去了刹车能力。

### 陷阱 3：`create_task()` 后忘了保存引用

```python
async def main():
    asyncio.create_task(background_work())   # 没有保存引用
    await asyncio.sleep(5)
```

Python 的垃圾回收不等人。如果你不保存 `create_task()` 返回的 Task 对象，它可能在完成任务之前就被 GC 回收，然后你会看到 `Task exception was never retrieved` 的警告。

**修复**：`task = asyncio.create_task(...)`，然后确保在适当的时候 await 它。

---

## 三、结构化并发：Python 3.11 的杀手锏

如果你只学一个 asyncio 新特性，就学 `TaskGroup`。

```python
import asyncio

async def do_fetch(url: str, results: list[str]):
    await asyncio.sleep(0.1)                     # 模拟 HTTP 请求
    results.append(f"fetched: {url}")

async def fetch_all(urls: list[str]) -> list[str]:
    results: list[str] = []
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            tg.create_task(do_fetch(url, results))
    return results

async def main():
    urls = [
        "https://example.com/a",
        "https://example.com/b",
        "https://example.com/c",
    ]
    data = await fetch_all(urls)
    print(data)

asyncio.run(main())
```

`TaskGroup` 做了三件事，之前你要手写几十行代码：

1. **退出 `async with` 块时自动等待所有子任务完成**——不再需要手动 `gather`
2. **任何一个子任务抛异常，所有兄弟任务都会被取消**——没有孤儿 task
3. **异常不会被吞掉**——用 `ExceptionGroup`（Python 3.11+ 内置）聚合所有错误

**为什么重要**：手动 `create_task` + `gather` 的模式，在异常路径下极容易产生"幽灵任务"——一个任务挂了，其他任务还在跑，永远没人 await 它们。TaskGroup 给了你确定性：要么全完成，要么全取消。

---

## 四、测试：你的 async 代码真的被测到了吗？

```python
# 这个测试函数永远不会被执行
async def test_fetch():
    result = await fetch("https://example.com")
    assert result is not None
```

不加 `@pytest.mark.asyncio`，pytest 不会去 await 你的 async 测试函数——它安安静静地 pass，你还以为一切正常。这是 async 测试最阴险的坑。

```python
import pytest
from unittest.mock import AsyncMock

async def fetch(url: str, client=None) -> str:
    if client:
        return await client.get(url)
    return f"real data from {url}"

@pytest.mark.asyncio
async def test_fetch():
    mock_client = AsyncMock()
    mock_client.get.return_value = "mocked data"

    result = await fetch("https://example.com", client=mock_client)
    assert result == "mocked data"
    mock_client.get.assert_awaited_once()
```

几个要点：

- **必须加 `@pytest.mark.asyncio`**，否则测试不会跑
- **用 `AsyncMock`**（Python 3.8+ 标准库自带）来 mock async 函数
- **pytest-asyncio >= 0.21 在 strict 模式下，每个测试默认新开一个 event loop**，保证隔离性。老旧版本的 `auto` 模式下行为不同，建议升级并在 `pytest.ini` 中设置 `asyncio_mode = strict`

**为什么重要**：静默跳过的测试比没有测试更危险。没有测试你知道要补；静默 pass 的测试给了你虚假的安全感。

---

## 五、一个缝合起来的实战例子

> **注意：本节代码需要 Python 3.11+**

把上面的知识串起来——一个带超时、限流、错误处理的并发 HTTP 爬虫骨架：

```python
import asyncio
import logging

logger = logging.getLogger(__name__)

SEM = asyncio.Semaphore(10)          # 最多 10 个并发请求


async def crawl(urls: list[str]) -> dict[str, str | BaseException]:
    """并发爬取多个 URL，带超时和限流控制。

    Returns:
        dict: 以 URL 为键，成功时值为响应字符串，失败时值为异常对象。
    """
    results: dict[str, str | BaseException] = {}

    try:
        async with asyncio.timeout(5.0):                    # 总超时 5 秒
            async with asyncio.TaskGroup() as tg:
                for url in urls:
                    tg.create_task(_fetch_one(url, results))
    except* TimeoutError:
        logger.warning("整体超时，返回已有结果")
    except* Exception as eg:
        logger.error(f"捕获到 {len(eg.exceptions)} 个异常")

    return results


async def _fetch_one(url: str, results: dict[str, str | BaseException]):
    async with SEM:                             # 限流 10 并发
        try:
            await asyncio.sleep(0.1)            # 模拟网络 I/O
            results[url] = f"data from {url}"
        except asyncio.CancelledError:
            # 被超时或兄弟异常取消
            results[url] = asyncio.CancelledError()
            raise                                # re-raise
        except Exception as e:
            # 记录网络错误、超时等实际异常
            logger.error(f"{url} 请求失败: {e}")
            results[url] = e


# 可运行演示
async def main():
    urls = [f"https://example.com/page/{i}" for i in range(20)]
    results = await crawl(urls)
    print(f"完成: {len(results)} 个请求")
    success = sum(1 for v in results.values() if isinstance(v, str))
    print(f"成功: {success}")

if __name__ == "__main__":
    asyncio.run(main())
```

输出示例：
```
完成: 20 个请求
成功: 20
```

这个例子展示了四个关键实践：
1. `TaskGroup` 确保结构化并发（无孤儿任务）
2. `asyncio.timeout()` 防止无限等待
3. `Semaphore` 做并发限流
4. `CancelledError` 正确处理（清理 + re-raise）

---

## 写在最后

async/await 是 Python I/O 并发工具箱里最强大的武器之一——但它有个残酷的前提：你必须真正理解它的运行模型。

三个带走的原则：

1. **`async def` 不提供魔法**。不替换阻塞调用，async 只会让你的代码更慢。
2. **用 `TaskGroup` 代替手动 task 管理**。结构化并发是你 sleep well at night 的保证。
3. **永远 re-raise `CancelledError`**。它是刹车，不是建议。

下次你打算把一个同步模块改成 async 时，先问自己：这里真的有大量 I/O 等待吗？如果答案是"否"——管住手，用同步就够了。

---

*参考来源：Python 3.14 官方文档 asyncio-dev、BBC Cloudfit asyncio 系列、Real Python Async IO Walkthrough、Python.org Async-SIG、pytest-asyncio 官方指南。*
