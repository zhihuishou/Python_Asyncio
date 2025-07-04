# Python_Asyncio

## 1.源码级爬虫系统架构导论

###### 框架设计：

（1）基于业务进行框架定制

（2）如果能搭建——技术相通

###### 技术储备：

（1）数据库知识

（2）异步编程

技术思路：
1.项目结构 2.请求管理和调度 3.请求下载 4.页面解析 5.数据存储 6.并发处理 7.日志记录 8.配置文件

## 2.系统架构

```python
异步编程：asyncio
1.定义协程函数
2.包装协程为任务
3.建立事件循环
```

```python
async def play():
	print("enter play")
    for _ in range(5):
        sleep_coro = asyncio.sleep(1)
        await asyncio.create_task(sleep_coro)
        for _ in range(1000):
            pass
async def main():
    print("enter main")
    start = time.time()
    tasks = []
    for _ in range(10):
        play_core = play()
        tasks.append(asyncio.create_task(play_coro))
    for t in tasks:
        await t
    print(f"total time:{time.time() - start} seconds")
    
main_coro = main()
asyncio.run(main_coro)
```

## 3.协程并发

主线程 >>> async库 >>> 协程并发任务（执行线程中并发代码）

1. **避免阻塞操作**：协程内不要用 `time.sleep()`，用 `await asyncio.sleep()`。
2. **I/O密集型场景**：适合网络请求、文件异步读写（如 `aiofiles`）。
3. **CPU密集型场景**：需配合 `run_in_executor` 使用线程池。

#### [1]asycio 模块

​	从PYTHON3.5引入的async和await，可以让coroutine的代码更简洁易读。

​	asyncio 被用于多个提供高性能python的异步框架基础，包括网络和网站服务。

```python
import asyncio

async def task(i):
     print(f"task {i} start")
     await asyncio.sleep(1) # 模拟IO事件
     print(f"task {i} end")

# 创建一个coroutine对象，也就是协程对象
if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    tasks = [task(1),task(2)]
    loop.run_until_complete(asyncio.wait(tasks))
    # loop.close()
```

#### [2]时间循环loop

调度协程的执行中枢

```python
loop = asyncio.get_event_loop()
loop.run_until_complete(my_coroutine())
loop.close()
```

#### [3]任务Task

对协程进行封装，并用于并发调度

```python
task = asyncio.create_task(my_coroutine())  # Python 3.7+
```



#### [4] asycio.ensure_future模块

 ensure_future是 Python 异步编程库 asyncio 中用于安排协程或 Future 对象执行的函数，其核心作用是将任何可等待对象（coroutine/Future）调度到事件循环中执行，即使未显式使用 `await` 也能保证任务被执行。 ‌

思考：

1.如果不使用ensure_future，那么你有多个协程想要并发执行（即同时运行），你需要将它们包装成任务。因为任务会被事件循环并发调度。如果不包装，那么用`await`依次等待每个协程就会变成串行执行。

2.在服务器场景中，每当有新连接到来时，我们通常希望为每个连接创建一个独立的任务来处理。如果不使用`ensure_future`或`create_task`，就无法动态地将新的协程作为任务加入事件循环。

3.：任务对象提供了更多的控制，比如取消（`cancel()`）、检查状态（`done()`）、添加回调（`add_done_callback()`）等。如果不将协程包装成任务，这些功能将无法使用。

举例当进行webAPI客户端的时候，假设不使用ensure_future任务调度，那么就变成串行执行

```python
# 使用 ensure_future 的版本
async def fetch_all(urls):
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    return await asyncio.gather(*tasks)

# 不使用的版本
async def fetch_all_serial(urls):
    results = []
    for url in urls:
        results.append(await fetch(url))  # 串行执行，性能低下
    return results
```



```python
import asyncio  

async def my_coroutine():  
    await asyncio.sleep(1)  
    print("Task complete")  

# 使用 ensure_future 调度协程  
asyncio.ensure_future(my_coroutine())  
```

思考：为什么数据抓取要实现异步逻辑？

1.提升效率，不需要等待单个任务结束后进行数据写入或者日志推送，那样效率太低

2.并发采集和多任务实现，处理密集型任务尤为重要

3.自写框架有便于理解底层构造，从而更好的进行源码优化

#### [5]应用场景

网络IO请求

```python
import aiohttp

async def fetch_url(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def main():
    urls = ["http://example.com", "http://example.org"]
    results = await asyncio.gather(*[fetch_url(url) for url in urls])
    for result in results:
        print(result)

asyncio.run(main())

```

#### [6]高级用法

6.1超时控制，使用wait_for 可以设置协程超时时间

```python
async def long_running_task():
    await asyncio.sleep(10)

async def main():
    try:
        await asyncio.wait_for(long_running_task(), timeout=5)
    except asyncio.TimeoutError:
        print("The task took too long!")

asyncio.run(main())

```

6.2取消任务：cancerlledError

```python
async def task():
    try:
        while True:
            print("Running...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("Task was cancelled")

async def main():
    t = asyncio.create_task(task())
    await asyncio.sleep(3)
    t.cancel()
    await t

asyncio.run(main())

```

6.3生产者消费者模式

```python
async def producer(queue):
    for i in range(5):
        await asyncio.sleep(1)
        await queue.put(i)
        print(f"Produced {i}")

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Consumed {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    producer_task = asyncio.create_task(producer(queue))
    consumer_task = asyncio.create_task(consumer(queue))

    await producer_task
    await queue.put(None)  # 用于通知消费者结束
    await consumer_task

asyncio.run(main())


```

#### [7]新版本asyncio

7.1通过create_task 以及gather函数，携程并发任务以及gather获取返回值

```python
def task01_callback(obj):
    print("obj:",obj)
    print("hello from task01_callback")
    print(obj.done(),obj.result())

async def work(i):
    print(f"Task {i} started")
    await asyncio.sleep(1)
    print(f"Task {i} finished")
    return i ** 2


async def main():
    start = time.time()
    works = [
        asyncio.create_task(work(1)), #卡住不要等，进行切换异步等待
        asyncio.create_task(work(2)),
    ]
    works[0].add_done_callback(task01_callback)
    # await asyncio.wait(works)
    ret = await asyncio.gather(*works)
    #异步等待的额时候会将所有返回结果收回，存到列表中，也就是return i**2
    print("ret:",ret)

    #为什么用gather？ 需要
    end = time.time()
    print(f"Total time: {end - start:.2f} se conds")


asyncio.run(main())
```

