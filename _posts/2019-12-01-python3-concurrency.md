---
layout: post
title: "Python3 Concurrency"
date: 2019-12-01 00:00:00 +0800
author: xiangxiang
categories: python
tags: [python asyncio aiohttp concurrency multiprocessing threading]
---
如何使用python3的并发

## 0x00 Overview
- python3中的三种并发方式

|      Type       | Switching Decision |  Processors |  python3 module |      When to Use     |
|:---------------:|:------------------:|:-----------:|:---------------:|:--------------------:|
| multithreading  |  Operation system  |      1      |    threading    |      I/O-bound       |
| multiprocessing |  Operation system  |     many    | multiprocessing |      CPU-bound       |  
| async IO        |   Programmer       |      1      |     asyncio     | I/O-bound(preferred) |

{% highlight text %}
`Use asyncio when you can, threading when you must`

asyncio的使用限制主要在于没有现成的库
{% endhighlight %}

## 0x01 asyncio in depth [PEP3156](https://www.python.org/dev/peps/pep-3156/)
- asyncio的后端实现方式 [参考](https://stackoverflow.com/questions/49005651/how-does-asyncio-actually-work/51116910#51116910)
{% highlight text %}
eventloop
   |
select waits(system call select/poll/epoll)
   |
I/O ready
   |
future.set_result() is called
   |
Task that added itself with add_done_callback() is now woken up
   |
Task calls .send() on the coroutine 
which goes all the way into the inner-most coroutine and wakes it up
   |
Data is being read from the buffer and returned to our humble user
{% endhighlight %}

- awaitable [PEP492](https://docs.python.org/3/library/asyncio-task.html#awaitables)
{% highlight text %}
We say that an object is an awaitable object if it can be used in an await expression.
Many asyncio APIs are designed to accept awaitables.

There are three main types of awaitable objects: coroutines, Tasks, and Futures.
{% endhighlight %}

- Coroutines/Futures/Tasks

`1. `[coroutines](https://docs.python.org/3/reference/datamodel.html#coroutines)

`**coroutines是基于generator的**` [PEP-342](https://www.python.org/dev/peps/pep-0342/)

{% highlight text %}
Coroutine objects are awaitable objects. 
A coroutine’s execution can be controlled by calling __await__() and iterating over the result. 

a) When the coroutine has finished executing and returns, the iterator raises StopIteration, 
and the exception’s value attribute holds the return value. 

b) If the coroutine raises an exception, it is propagated by the iterator. 

c) Coroutines should not directly raise unhandled StopIteration exceptions.
{% endhighlight %}




`2. `[Futures](https://docs.python.org/3/library/asyncio-future.html#asyncio.Future)
{% highlight text %}
A Future represents an eventual result of an asynchronous operation. 

a) Not thread-safe.

b) Future is an awaitable object. 

c) Coroutines can await on Future objects until they either have a result or an exception set, 
or until they are cancelled.

d) Typically Futures are used to enable low-level callback-based code 
(e.g. in protocols implemented using asyncio transports) to 
interoperate with high-level async/await code.

e) The rule of thumb is to never expose Future objects in user-facing APIs, 
and the recommended way to create a Future object is to call loop.create_future(). 

f) This way alternative event loop implementations can inject 
their own optimized implementations of a Future object.
{% endhighlight %}

`3. `[Tasks](https://docs.python.org/3/library/asyncio-task.html#task-object)
{% highlight text %}
Tasks are used to schedule coroutines concurrently

a) If a coroutine awaits on a Future, 
the Task suspends the execution of the coroutine and waits for the completion of the Future. 
When the Future is done, the execution of the wrapped coroutine resumes.

b) Event loops use cooperative scheduling: an event loop runs one Task at a time.
While a Task awaits for the completion of a Future, 
the event loop runs other Tasks, callbacks, or performs IO operations.
{% endhighlight %}

## 0x02 asyncio in action
基本用法主要参考[python官方文档](https://docs.python.org/3/library/asyncio-task.html)

step 1. 首先构建小的coro

step 2. 通过小的coro构建task, task可以是多个(比如一个producer task,一个customer task)

step 3. 选择task的运行模式(chained model, producer/customer model)

step 4. run

- `async def`: Native Coroutine/Asynchronous Generator declaration syntax

`1. `**generator-based coroutines**未来不会再支持 [两者的区别](https://www.python.org/dev/peps/pep-0492/#coroutine-objects)
{% highlight python %}
import asyncio

@asyncio.coroutine
def py34_coro():
    """Generator-based coroutine, older syntax"""
    yield from stuff()

async def py35_coro():
    """Native coroutine, modern syntax"""
    await stuff()
{% endhighlight %}
`2. ` Asynchronous Generator是py36新特性,不常用先不讨论 [PEP525](https://www.python.org/dev/peps/pep-0525/)
{% highlight python %}
# OK - this is an async generator
async def g(x):
    yield x  
{% endhighlight %}

- Things a coroutine can do:
{% highlight python %}
# suspends the coroutine until the future is done, then returns the future's result, 
# or raises an exception, which will be propagated. 
# (If the future is cancelled, it will raise a CancelledError exception.) 
# Note that tasks are futures, and everything said about futures also applies to tasks.
result = await future

# wait for another coroutine to produce a result 
# (or raise an exception, which will be propagated). 
# The coroutine expression must be a call to another coroutine.
result = await coroutine

# produce a result to the coroutine that is waiting for this one using await
return expression

# raise an exception in the coroutine that is waiting for this one using await
raise exception 
{% endhighlight %}

- Tasks

`1. ` Creating tasks

{% highlight python %}
# In Python 3.7+
task = asyncio.create_task(coro())
...

# This works in all Python versions but is less readable
task = asyncio.ensure_future(coro())
{% endhighlight %}

`2. `Running Tasks Concurrently
{% highlight python %}
# awaitable asyncio.gather(*aws, loop=None, return_exceptions=False)
# loop参数会被弃用

async def _main():
    # Schedule three calls *concurrently*:
    await asyncio.gather(
        task1(),
        task2(),
        task3(),
    )

# run
asyncio.run(_main())

# old style
# loop = asyncio.get_event_loop()
# loop.run_until_complete(_main())
{% endhighlight %}

- Design Patterns (code snippets)
这里的design pattern是对于task类型而言的

|       类型       |           特点                   |
|:---------------:|:--------------------------------:|     
|     chained     | eventloop中的每个task都是一样的顺序执行, task中会有大量的io操作|
|producer/customer| eventloop中的有两(多)种不同类型的task，task中会有大量的io操作 |



`1. ` chained model

每个task都是先运行phase1(io操作)，等待phase1返回结果后运行phase2(io操作)

code from [pymotw](https://pymotw.com/3/asyncio/coroutines.html)

{% highlight python %}
import asyncio

async def outer():
    print('in outer')
    print('waiting for result1')
    result1 = await phase1()
    print('waiting for result2')
    result2 = await phase2(result1)
    return (result1, result2)

async def phase1():
    print('in phase1')
    return 'result1'

async def phase2(arg):
    print('in phase2')
    return 'result2 derived from {}'.format(arg)

event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(outer())
    print('return value: {!r}'.format(return_value))
finally:
    event_loop.close()
{% endhighlight %}

`2. ` producer/customer model

task中有两类: 生产者和消费者
难点在于通知消费者生产已经完成了，在示例代码中是在producer完成之后先调用`await q.join()`等待队列消费完成，后显示对每个customer调用`cancel`

code from [realpython](https://realpython.com/async-io-python/)

{% highlight python %}
import asyncio
import itertools as it
import os
import random
import time

async def makeitem(size: int = 5) -> str:
    return os.urandom(size).hex()

async def randsleep(a: int = 1, b: int = 5, caller=None) -> None:
    i = random.randint(0, 10)
    if caller:
        print(f"{caller} sleeping for {i} seconds.")
    await asyncio.sleep(i)

async def produce(name: int, q: asyncio.Queue) -> None:
    n = random.randint(0, 10)
    for _ in it.repeat(None, n):  # Synchronous loop for each single producer
        await randsleep(caller=f"Producer {name}")
        i = await makeitem()
        t = time.perf_counter()
        await q.put((i, t))
        print(f"Producer {name} added <{i}> to queue.")

async def consume(name: int, q: asyncio.Queue) -> None:
    while True:
        await randsleep(caller=f"Consumer {name}")
        i, t = await q.get()
        now = time.perf_counter()
        print(f"Consumer {name} got element <{i}>"
              f" in {now-t:0.5f} seconds.")
        q.task_done()

async def main(nprod: int, ncon: int):
    q = asyncio.Queue()
    producers = [asyncio.create_task(produce(n, q)) for n in range(nprod)]
    consumers = [asyncio.create_task(consume(n, q)) for n in range(ncon)]
    await asyncio.gather(*producers)
    await q.join()  # Implicitly awaits consumers, too
    for c in consumers:
        c.cancel()

if __name__ == "__main__":
    import argparse
    random.seed(444)
    parser = argparse.ArgumentParser()
    parser.add_argument("-p", "--nprod", type=int, default=5)
    parser.add_argument("-c", "--ncon", type=int, default=10)
    ns = parser.parse_args()
    start = time.perf_counter()
    asyncio.run(main(**ns.__dict__))
    elapsed = time.perf_counter() - start
    print(f"Program completed in {elapsed:0.5f} seconds.")
{% endhighlight %}

- optimization: taskpool

asyncio的task数量在实际中会受到限制，可能原因如下：

`1. `可以一起创建task数量有限：实际中不可能在程序一开始就生成无限数量的task

`2. `eventloop中并发的task数量有限：同样是受到资源的限制

引入TaskPool [参考链接](https://medium.com/@cgarciae/making-an-infinite-number-of-requests-with-python-aiohttp-pypeln-3a552b97dc95)

核心思想是使用信号量`asyncio.Semaphore`限制task的生成数量,
一个task完成后释放信号量使得新的一个task准备并执行

{% highlight python %}
# TaskPool is a Asynchronous Context Manager 
# https://www.python.org/dev/peps/pep-0492/#asynchronous-context-managers-and-async-with
# Two new magic methods are added: __aenter__ and __aexit__. Both must return an awaitable.
# Remember: We say that an object is an awaitable object if it can be used in an await expression.
class TaskPool(object):
    def __init__(self, workers):
        self._semaphore = asyncio.Semaphore(workers)
        self._tasks = set()

    async def put(self, coro):  
        await self._semaphore.acquire()
        task = asyncio.ensure_future(coro)
        self._tasks.add(task)
        task.add_done_callback(self._on_task_done)

    def _on_task_done(self, task):
        self._tasks.remove(task)
        self._semaphore.release()

    async def join(self):
        await asyncio.gather(*self._tasks)

    async def __aenter__(self):
        return self

    def __aexit__(self, exc_type, exc, tb):
        return self.join()
{% endhighlight %}


Code snippets for taskpool
{% highlight python %}
tasks_limit = 10

async def _main(total_task_numbers):
    async with TaskPool(tasks_limit) as tasks
        for i in range(total_task_numbers):
            await tasks.put(task())

# old style
# loop = asyncio.get_event_loop()
# loop.run_until_complete(_main())

# new style
asyncio.run(_main())
{% endhighlight %}