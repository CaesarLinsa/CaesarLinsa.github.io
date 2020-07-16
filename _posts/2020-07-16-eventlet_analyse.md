---
layout: post
title: 'Eventlet协程库分析'
subtitle:   "Eventlet conroutine analysis"
date:       2020-07-16 21:39:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - python
---
## Eventlet
文中所有注释见我的[github](https://github.com/CaesarLinsa/Eventlet)    
python中由于使用GIL，每个CPU在同一时间只能执行一个线程。在单进程的多线程下, 即使是多核CPU也仍然是单线程。使用Eventlet协程，可以实现线程中协程调度，
更重要的是，协程切换，资源消耗比线程小。eventlet依赖greenlet库（C语言协程库，提供python接口），其调度，需要手动switch切换。在eventlet中使用Hub进行调度。
```python
class BaseHub(object):

    READ = READ
    WRITE = WRITE

    def __init__(self, clock=time.time):
        self.listeners = {READ:{}, WRITE:{}}
        self.secondaries = {READ:{}, WRITE:{}}

        self.clock = clock
        # 父协程
        
        self.greenlet = greenlet.greenlet(self.run)
        self.stopping = False
        self.running = False
        # 存储Timers，时间到时，调度子协程
        
        self.timers = []
        self.next_timers = []
        # 文件读写监听类
        
        self.lclass = FdListener
        self.debug_exceptions = True
```
### spawn协程
使用spawn创建GreenThread，即greenlet.greenlet协程。
```python
>>> import eventlet
>>> def test():
...     print("hello world")
...
>>> eventlet.spawn(test)
<eventlet.greenthread.GreenThread object at 0x000001CD0453B468>
>>> eventlet.sleep(0.1)
hello world
>>>
```
spawn 创建一个协程，如下: 初始化Hub，创建GreenThread协程，将Hub的协程作为父协程。
`hub.schedule_call_global` 将协程调度方法g.switch(func, args, kargs)注册在hub的next_times中。
```python
# greenthread.py

def spawn(func, *args, **kwargs):
    """Create a greenthread to run ``func(*args, **kwargs)``.  Returns a 
    :class:`GreenThread` object which you can use to get the results of the 
    call.
    
    Execution control returns immediately to the caller; the created greenthread
    is merely scheduled to be run at the next available opportunity.  
    Use :func:`spawn_after` to  arrange for greenthreads to be spawned 
    after a finite delay.
    """
    hub = hubs.get_hub()
    # hub.greenlet 父协程
    
    g = GreenThread(hub.greenlet)
    # hub中注册self.next_timers(hub初始化时间+ 0，timer对象)
    
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    return g
```
**eventlet.sleep**，释放当前main协程对cpu占用(greenlet默认当前执行环境便是一个main协程)

```python
# greenthread.py

def sleep(seconds=0):
    """Yield control to another eligible coroutine until at least *seconds* have
    elapsed.

    *seconds* may be specified as an integer, or a float if fractional seconds
    are desired. Calling :func:`~greenthread.sleep` with *seconds* of 0 is the
    canonical way of expressing a cooperative yield. For example, if one is
    looping over a large list performing an expensive calculation without
    calling any socket methods, it's a good idea to call ``sleep(0)``
    occasionally; otherwise nothing else will run.
    """
    # 获取单例的hub对象

    hub = hubs.get_hub()
    assert hub.greenlet is not greenlet.getcurrent(), 'do not call blocking functions from the mainloop'
    # 注册经过seconds，从父协程切换到main协程的方法
    
    timer = hub.schedule_call_global(seconds, greenlet.getcurrent().switch)
    try:
        # 调用switch方法（方法会切换到父协程，执行其run方法，遍历调度所有注册的协程方法
        
        hub.switch()
    finally:
        timer.cancel()

```

```python
# hubs/hub.py

def switch(self):
    cur = greenlet.getcurrent()
    assert cur is not self.greenlet, 'Cannot switch to MAINLOOP from MAINLOOP'
    switch_out = getattr(cur, 'switch_out', None)
    if switch_out is not None:
        try:
            switch_out()
        except:
            self.squelch_generic_exception(sys.exc_info())
            clear_sys_exc_info()
    if self.greenlet.dead:
        self.greenlet = greenlet.greenlet(self.run)
    try:
        if self.greenlet.parent is not cur:
            cur.parent = self.greenlet
    except ValueError:
        pass  
    # gets raised if there is a greenlet parent cycle
    
    clear_sys_exc_info()
    # 执行self.run，进行调度表
    
    return self.greenlet.switch()
```
父协程是一个死循环，意味着父协程对子协程调度一直存在。当切换到父协程时（即循环中），便会调度执行注册的子协程（包括sleep时，注册的协程，在等待长时间被切换回main协程，继续向下执行）。
当main协程结束时，整个程序结束。
```python
# hubs/hub.py

def run(self):
    """Run the runloop until abort is called.
    """
    if self.running:
        raise RuntimeError("Already running!")
    try:
        self.running = True
        self.stopping = False
        while not self.stopping:
            # self.timers中注册时间，从小到大进行堆排序
            
            self.prepare_timers()
            # self.clock()获取当前时间，通过对比注册协程时间
            
            # 若小于等于则立刻调用，否则退出，等待到时间再调用
            
            self.fire_timers(self.clock())
            self.prepare_timers()
            wakeup_when = self.sleep_until()
            if wakeup_when is None:
                sleep_time = self.default_sleep()
            else:
                sleep_time = wakeup_when - self.clock()
            # wait存在注册读写事件时，使用epoll.poll阻塞进程，等待连接接入
            
            # 否则sleep指定时间
            
            if sleep_time > 0:
                self.wait(sleep_time)
            else:
                self.wait(0)
        else:
            del self.timers[:]
            del self.next_timers[:]
    finally:
        self.running = False
        self.stopping = False

```
## WSGI server
一个使用WSGI server的例子，如下
```python
import eventlet
from eventlet import wsgi


def hello_world(env, start_response):
    if env['PATH_INFO'] != '/':
        start_response('404 Not Found', [('Content-Type', 'text/plain')])
        return ['Not Found\r\n']
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Hello, World!\r\n']


wsgi.server(eventlet.listen(('', 8090)), hello_world)
```
`eventlet.listen` 创建socket对象，在wsgi.server中，accept接收用户连接。
```python
# wsgi.py :: server

# 上略，创建Server(BaseHTTPServer.HTTPServer)和一个协程池（用来处理请求）

while True:
    try:
        # 此处有魔法
        
        client_socket = sock.accept()
        try:
            # 孵化一个请求处理协程，注册在Hub中
            
            pool.spawn_n(serv.process_request, client_socket)
        except AttributeError:
            warnings.warn("wsgi's pool should be an instance of " \
                "eventlet.greenpool.GreenPool, is %s. Please convert your"\
                " call site to use GreenPool instead" % type(pool),
                DeprecationWarning, stacklevel=2)
            pool.execute_async(serv.process_request, client_socket)
    except ACCEPT_EXCEPTIONS, e:
        if get_errno(e) not in ACCEPT_ERRNO:
            raise
    except (KeyboardInterrupt, SystemExit):
        serv.log.write("wsgi exiting\n")
        break
        
```
`client_socket = sock.accept()`是一个魔法方法，非阻塞读取，其内部是一个循环。
当用户接入时，返回连接对象，否则返回None
```python
# greenio.py :: GreenSocket
def accept(self):
    if self.act_non_blocking:
        return self.fd.accept()
    fd = self.fd
    while True:
        # 无用户连接时连接失败，Resource temporarily unavailable，进行trampoline
        
        res = socket_accept(fd)
        # 用户连接时，res非None，返回type(self)(client), addr
        
        # 从而孵化一个协程。wsgi.py :: server循环，调用sock.accept()，因为无连接返回空，
        
        # 调用trampoline方法，注册读事件和切换main协程方法，其中hub.switch切换到父协程执行刚才孵化的处理请求协程
        
        if res is not None:
            client, addr = res
            set_nonblocking(client)
            return type(self)(client), addr
        trampoline(fd, read=True, timeout=self.gettimeout(),
                       timeout_exc=socket.timeout("timed out"))
```

```python
# hubs/__init__.py

def trampoline(fd, read=None, write=None, timeout=None, 
               timeout_exc=timeout.Timeout):
    """Suspend the current coroutine until the given socket object or file
    descriptor is ready to *read*, ready to *write*, or the specified
    *timeout* elapses, depending on arguments specified.

    To wait for *fd* to be ready to read, pass *read* ``=True``; ready to
    write, pass *write* ``=True``. To specify a timeout, pass the *timeout*
    argument in seconds.

    If the specified *timeout* elapses before the socket is ready to read or
    write, *timeout_exc* will be raised instead of ``trampoline()``
    returning normally.
    
    .. note :: |internal|
    """
    t = None
    hub = get_hub()
    current = greenlet.getcurrent()
    assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
    assert not (read and write), 'not allowed to trampoline for reading and writing'
    try:
        fileno = fd.fileno()
    except AttributeError:
        fileno = fd
    if timeout is not None:
        t = hub.schedule_call_global(timeout, current.throw, timeout_exc)
    try:
        if read:
            # 注册读监听（此处是"main"协程), 用于跳出父协程, 切换回"main"协程
            
            # 在epoll多路复用时，一旦监听到读事件
            
            # 执行current.switch切换到hub 父协程的switch方法，到run方法
            
            listener = hub.add(hub.READ, fileno, current.switch)
        elif write:
            listener = hub.add(hub.WRITE, fileno, current.switch)
        try:
            # 调用hub的switch，到父协程的run方法
            
            # 从而父协程进行调度
            
            return hub.switch()
        finally:
        # 读事件时，切换"mian"协程到此处
        
            hub.remove(listener)
    finally:
        if t is not None:
            t.cancel()
```
hub.switch和run方法见上spawn中所示，对子协程调度完成后，调用wait方法。
父协程`self.poll.poll`等待用户连接(读事件)，建立后，在`readers.get(fileno, noop).cb(fileno)`
切回main协程的**trampoline** `finally`方法，执行完成后，accept循环继续调用，建立连接。

```python
# hubs/poll.py

def wait(self, seconds=None):
    readers = self.listeners[READ]
    writers = self.listeners[WRITE]

    if not readers and not writers:
        if seconds:
            sleep(seconds)
        return
    try:
        # 阻塞，等待读事件
        
        presult = self.poll.poll(int(seconds * self.WAIT_MULTIPLIER))
    except select.error, e:
        if get_errno(e) == errno.EINTR:
            return
        raise
    SYSTEM_EXCEPTIONS = self.SYSTEM_EXCEPTIONS

    for fileno, event in presult:
        try:
            if event & READ_MASK:
                # current.switch 此处切回"main"协程
                readers.get(fileno, noop).cb(fileno)
            if event & WRITE_MASK:
                writers.get(fileno, noop).cb(fileno)
            if event & select.POLLNVAL:
                self.remove_descriptor(fileno)
                continue
            if event & EXC_MASK:
                readers.get(fileno, noop).cb(fileno)
                writers.get(fileno, noop).cb(fileno)
        except SYSTEM_EXCEPTIONS:
            raise
        except:
            self.squelch_exception(fileno, sys.exc_info())
            clear_sys_exc_info()

```
建立连接，孵化协程。accept循环调用，socket_accept(fd)返回None，从而**trampoline** 中hub.switch切换父协程处理。
如此每次请求，完成一次`父协程-->main协程-->父协程`。    
总而言之，main协程建立用户连接，孵化出子协程，然后切换回父协程处理，父协程处理完成后，等待用户连接后，切换到main协程进行建立连接，孵化子协程，切换父协程处理

