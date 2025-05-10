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

await 的工作机制：

1. 挂起当前协程
2. 将控制权交还给事件循环
3. 当 awaitable 完成时，恢复协程执行
4. 返回 awaitable 的结果或引发异常
### 1.3 事件循环的角色
事件循环是 asyncio 的核心，负责：

- 调度协程执行
- 处理 IO 操作
- 管理回调
- 运行直到所有任务完成
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
