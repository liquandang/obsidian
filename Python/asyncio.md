## 一、async/await 深度解析
### 1.1 协程函数与协程对象
协程函数是用 `async def` 定义的函数，调用它会返回一个协程对象：
```PYTHON
async def coro_func():
    return "result"

# 调用协程函数不会执行代码，而是返回协程对象
coro_obj = coro_func()
print(type(coro_obj))  # <class 'coroutine'>
print(inspect.iscoroutine(coro_obj)) #true
```
协程对象有以下重要特性：

- 必须被事件循环执行或通过 await 等待
- 不能被直接调用或 yield from
- 可以通过 `inspect.iscoroutine()` 检测
### 1.2 await 表达式详解
await 表达式会暂停当前协程的执行，直到 awaitable 对象完成：
```PYTHON
async def main():
    print("Start")
    result = await coro_func()  # 暂停点
    print("Got:", result)
```

#### await 的工作机制：

1. 挂起当前协程
2. 将控制权交还给事件循环
3. 当 awaitable 完成时，恢复协程执行
4. 返回 awaitable 的结果或引发异常

#### **`await` 的关键步骤**

1. **挂起协程**：当执行到 `await` 时，当前协程会立即暂停，并返回一个 **可等待对象**（如 `Future`）。
2. **注册回调**：事件循环会监听这个 `Future`，并在它完成时恢复协程。
3. **让出控制权**：事件循环转而执行其他任务。

### 1.3 事件循环的角色
事件循环是 asyncio 的核心，负责：

- 调度协程执行
- 处理 IO 操作
- 管理回调
- 运行直到所有任务完成

#### **事件循环的作用**

- 事件循环是异步编程的核心，负责：
    
    1. **调度协程**：决定哪个协程在何时执行。
    2. **监听 I/O 事件**：当某个 I/O 操作（如网络请求）完成时，唤醒对应的协程。

#### **工作流程**

1. 协程通过 `await` 暂停时，事件循环会记录它的状态（如“等待某个 Socket 数据”）。
2. 事件循环继续执行其他就绪的协程。
3. 当 I/O 操作完成（如收到网络响应），事件循环将对应的协程重新加入执行队列。

```python
# 伪代码
while True:
    # 检查哪些 I/O 操作已完成（通过操作系统 select/poll/epoll）
    ready_events = poll_io_events()
    
    # 执行这些事件关联的回调（恢复协程）
    for event in ready_events:
        event.callback()

    # 执行就绪的协程
    run_ready_coroutines()
```
   
```python
async def main():
    await asyncio.sleep(1)

# 获取事件循环的三种方式
loop = asyncio.get_event_loop()       # 1. 获取当前循环
loop = asyncio.new_event_loop()       # 2. 创建新循环
loop = asyncio.get_running_loop()     # 3. 获取正在运行的循环(必须在协程内调用)
```


## 二、可等待对象(Awaitables)深度解析
可等待协程对象、task、future

### 2.1 协程对象(Coroutine)
```python
async def nested():
    return 42

async def main():
    # 直接await协程
    print(await nested())  # 输出42
    
    # 协程执行流程
    coro = nested()
    result = await coro
    print(result)  # 输出42
```
### 2.2 Task 对象

Task 是协程的包装器，用于并发执行：
```python
async def task_demo():
    # 创建Task的三种方式
    task1 = asyncio.create_task(nested())  # Python 3.7+
    task2 = asyncio.ensure_future(nested())  # 旧版兼容
    task3 = asyncio.get_event_loop().create_task(nested())  # 显式使用loop
    
    # Task状态查询
    print(task1.done())  # False
    await task1
    print(task1.done())  # True
    print(task1.result())  # 42
```
Task 的重要特性：

- 创建后自动调度执行
- 可取消(`task.cancel()`)
- 可添加完成回调
- 可查询状态和结果
### 2.3 Future 对象

Future 是更底层的可等待对象：
```python
async def future_demo():
    loop = asyncio.get_running_loop()
    print(loop) #<_UnixSelectorEventLoop running=True closed=False debug=False> 
    future = loop.create_future()
    
    # 设置结果回调
    def set_result():
        print("Setting result")
        future.set_result("done")
    
    # 2秒后设置结果
    loop.call_later(2, set_result)
    
    # 等待Future完成
    result = await future
    print(result)  # 输出"done"
```
Future 的关键方法：

- `set_result(result)` - 标记成功完成
- `set_exception(exception)` - 标记失败
- `cancel()` - 取消Future
- `add_done_callback(cb)` - 添加回调
## 三、concurrent.futures 深度集成
### 3.1 ThreadPoolExecutor 与 asyncio

将阻塞IO操作转移到线程池：
```python
import asyncio  
from concurrent.futures import ThreadPoolExecutor  
  
import requests  
  
def blocking_io(url):  
    # 模拟阻塞IO  
    return requests.get(url).content.decode('utf8')  
  
  
async def fetch_urls():  
    urls = ["https://baidu.com", "https://baijiahao.baidu.com/s?id=1831698668695775055"]  
  
    with ThreadPoolExecutor(max_workers=5) as pool:  
        loop = asyncio.get_running_loop()  
  
        # 创建Future列表  
        futures = [  
            loop.run_in_executor(pool, blocking_io, url)  
            for url in urls  
        ]  
  
        # 等待所有Future完成  
        #这里的 `*futures` 表示将 `futures` 列表中的每个 `Future` 对象解包，作为 `asyncio.gather()` 的单独参数传递进去。 等价：results = await asyncio.gather(futures[0], futures[1], ..., futures[n])
        results = await asyncio.gather(*futures)  
        print(results, )  
        return results  
  
asyncio.run(fetch_urls())
```

### 3.2 ProcessPoolExecutor 与 asyncio

CPU密集型任务使用进程池：
```python
import asyncio  
from concurrent.futures import ProcessPoolExecutor  
  
  
def cpu_bound(n):  
    return sum(i * i for i in range(n))  
  
  
async def compute_sums():  
    numbers = [6, 7, 8]  
  
    with ProcessPoolExecutor() as pool:  
        loop = asyncio.get_running_loop()  
        tasks = [  
            loop.run_in_executor(pool, cpu_bound, n)  
            for n in numbers  
        ]  
        return await asyncio.gather(*tasks)  
  
  
if __name__ == "__main__":  
    results = asyncio.run(compute_sums())  
    print(results)
```

## 3.3ProcessPoolExecutor和ThreadPoolExecutor区别

| 特性         | `ProcessPoolExecutor` (进程池)    | `ThreadPoolExecutor` (线程池) |
| ---------- | ------------------------------ | -------------------------- |
| **底层机制**   | 使用 **多进程** (`multiprocessing`) | 使用 **多线程** (`threading`)   |
| **内存隔离**   | 每个进程有独立内存（数据不共享）               | 所有线程共享同一进程内存               |
| **GIL 影响** | **不受 GIL 限制**（适合 CPU 密集型任务）    | **受 GIL 限制**（适合 I/O 密集型任务） |
| **启动开销**   | 较高（进程创建较慢）                     | 较低（线程创建较快）                 |
| **数据共享**   | 需用 `Queue`、`Pipe` 或共享内存        | 可直接共享变量（但需线程安全）            |
| **适用场景**   | CPU 密集型计算（如数学计算、图像处理）          | I/O 密集型任务（如网络请求、文件读写）      |
### 3.4 Future 转换机制详解

`asyncio.wrap_future()` 的内部工作原理：
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def blocking_io():
    import time
    time.sleep(1)
    return "IO result"

async def main():
    loop = asyncio.get_running_loop()
    
    # 1. 创建线程池
    with ThreadPoolExecutor() as pool:
        # 2. 提交任务到线程池，获得 concurrent.futures.Future
        con_future = pool.submit(blocking_io)
        
        # 3. 转换为 asyncio.Future
        asyncio_future = asyncio.wrap_future(con_future)
        
        # 4. 现在可以 await 了
        result = await asyncio_future
        print(result)  # 输出: IO result

asyncio.run(main())
```

## 四、异步上下文管理
异步上下文管理器通过 `async with` 语句使用，它需要实现两个特殊方法：

- `__aenter__()` - 异步进入上下文
- `__aexit__()` - 异步退出上下文
```python
import asyncio

class AsyncContextManager:
    async def __aenter__(self):
        print("Entering context")
        await asyncio.sleep(1)
        return self  # 可以返回有用的对象
    
    async def __aexit__(self, exc_type, exc, tb):
        print("Exiting context")
        await asyncio.sleep(1)
        # 可以处理异常，返回True表示抑制异常

async def main():
    async with AsyncContextManager() as manager:
        print("Inside context")

asyncio.run(main())
```

1. 异步上下文管理器必须与 `async with` 一起使用
2. `__aenter__` 和 `__aexit__` 必须是协程函数（使用 `async def`）
3. 在 `async with` 块内部可以正常使用 `await`
4. Python 3.10+ 支持在异步上下文管理器中使用 `except` 和 `finally` 块

异步上下文管理器是管理异步资源（如数据库连接、网络连接、锁等）的理想工具，可以确保资源被正确获取和释放。
```python
import asyncio  
  
import aiohttp  
  
async def fetch_url(url: str):  
    connector = aiohttp.TCPConnector(ssl=False)  # 禁用SSL验证  
  
    async with aiohttp.ClientSession(connector = connector) as session:  
        async with session.get(url) as response:  
            return await response.text()  
  
async def main():  
    context = await fetch_url("https://blog.csdn.net/weixin_40544270/article/details/147751411")  
    print(context)  
  
if __name__ == '__main__':  
    asyncio.run(main())
```


Python 3.7+ 提供了更简洁的方式来创建异步上下文管理器：
```python
from contextlib import asynccontextmanager
import asyncio

@asynccontextmanager
async def async_context_manager():
    print("Entering context")
    await asyncio.sleep(1)
    try:
        yield "some value"  # 这个值会被赋给as后面的变量
    finally:
        print("Exiting context")
        await asyncio.sleep(1)

async def main():
    async with async_context_manager() as value:
        print(f"Inside context, got: {value}")

asyncio.run(main())
```

我们来看一个数据库的例子：
```python
@asynccontextmanager
async def get_db_connection():
    conn = await async_connect_to_db()
    try:
        yield conn
    finally:
        await conn.close()

async def query_db():
    async with get_db_connection() as conn:
        result = await conn.execute("SELECT * FROM table")
        return result
```

