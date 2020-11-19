---
layout: post
title: 'python3.7原生协程'
subtitle:   "python3.7 asyncio"
date:       2020-11-19 23:00:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - Vue
---
### 协程
从python3.4到最终python3.7， 将asyncio合入到最终版本。在之前版本，都在使用第三方包eventlet，gevent等协成库。但大体思路是一样的    
**事件循环** : 将协程加入到事件列表中(堆)中，协程依次执行，当协程短暂阻塞后，重新唤醒。    
**Future对象**: 协程的载体，用于表示协程执行的对象（类似与js中promise对象）    
**Task对象**:  继承Future对象，协程的载体    
**协程**: 执行具体某个异步任务的函数。函数体中的 await 关键字可以将协程的控制权释放给事件循环。    

```python
import asyncio
async def foo():
    print('Running in foo')
    await asyncio.sleep(0)
    print('Explicit context switch to foo again')
# 创建事件循环    
ioloop = asyncio.get_event_loop() 
# 注册协程并运行    
ioloop.run_until_complete(foo())
# 关闭事件循环    
ioloop.close()
```
单个协程，注册到事件循环中，甚至没有同步执行快。协程主要用在有频繁io操作且比较慢，此时可以释放cpu 时间片，让其他协程执行，比如是很慢的request请求。
但如果是相应比较快的request，由于切换协程消耗的时间大于请求阻塞时间，使用协程反而会比较慢。

多个任务的协程，如下。 await修饰的必须是awaitable对象或者使用async和await装饰的方法，其实方法这样套await，最终还是要返回一个自己封装的awaitable对象。
目前可以使用的已知[仓库](https://github.com/aio-libs)
```python
import asyncio

async def foo():
    print('Running in foo')
    await asyncio.sleep(0)
    print('Explicit context switch to foo again')

async def bar():
    print('Explicit context to bar')
    await asyncio.sleep(0)
    print('Implicit context switch back to bar')

ioloop = asyncio.get_event_loop()
# 任务列表    
tasks = [ioloop.create_task(foo()), ioloop.create_task(bar())]
# 将list 协程化（返回done和pending 协成set）    
wait_tasks = asyncio.wait(tasks)
# 注册执行    
ioloop.run_until_complete(wait_tasks)
ioloop.close()
```
### 对比tornado
对比tornado中使用@gen.coroutine装饰，yield释放cpu，貌似更方便，因为其为一个普通方法。
操作更简单
```python
from tornado import gen
from tornado import ioloop
@gen.coroutine
def foo():
    print('Running in foo')
    yield gen.sleep(0)
    print('Explicit context switch to foo again')

@gen.coroutine
def bar():
    print('Explicit context to bar')
    yield gen.sleep(0)
    print('Implicit context switch back to bar')

@gen.coroutine
def main():
    yield [foo(), bar()]

ioloop = ioloop.IOLoop.current()
ioloop.run_sync(main)
```

