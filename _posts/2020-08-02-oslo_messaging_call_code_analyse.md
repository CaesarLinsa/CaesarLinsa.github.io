---
layout: post
title: 'oslo.messaging RPC调用分析'
subtitle:   "oslo.messaging RPC call analyse"
date:       2020-08-02 16:40:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - openstack
---
## oslo.messaging RPC client-server调用实现
(以rabbitmq实现为例)
可以参看[示例代码](https://caesarlinsa.github.io/2020/07/07/example_for_oslo_messaging_usage/)
### server端
oslo.messaging server端启动时,self.dispatcher为RPCDispatcher类，RPCDispatcher在`__call__`中，
完成调用endpoint中method

```python
if self._executor is not None:
    return

try:
    listener = self.transport._listen(self.target)
except driver_base.TransportDriverError as ex:
    raise ServerListenError(self.target, ex)

self._executor = self._executor_cls(self.conf, listener,
                                    self.dispatcher)
self._executor.start()
```
因此在`self._executor.start()` 调用`self.listener.poll()`

```python
def start(self):
    self._running = True
    while self._running:
        self._dispatch(self.listener.poll())
```
在poll时，对目前所有队列监听，如果监听到消息，将其初始化为AMQPIncomingMessage类，
放入self.incoming中(原因第二节)
```python
def poll(self):
    while True:
        if self.incoming:
            return self.incoming.pop(0)
        self.conn.consume(limit=1)
```

```python
def _dispatch(self, incoming):
    try:
        # callback 为 self.dispatcher即RPCDispatcher对象
        
        # 此处调用其__call__方法， incoming为AMQPIncomingMessage类
        
        reply = self.callback(incoming.ctxt, incoming.message)
        # 如果返回值非空，将结果发送
        
        if reply:
            incoming.reply(reply)
    except messaging.ExpectedException as e:
        _LOG.debug('Expected exception during message handling (%s)' %
                   e.exc_info[1])
        incoming.reply(failure=e.exc_info, log_failure=False)
    except Exception:
        # sys.exc_info() is deleted by LOG.exception().
        
        exc_info = sys.exc_info()
        _LOG.error("Failed to process message... skipping it.",
                   exc_info=exc_info)
        incoming.reply(failure=exc_info)
```
以reply_q 或者msg_id将结果消息返回
```python
def reply(self, reply=None, failure=None, log_failure=True):
    with self.listener.driver._get_connection() as conn:
        self._send_reply(conn, reply, failure, log_failure=log_failure)
        self._send_reply(conn, ending=True)
```
### AMQPIncomingMessage 放入self.incoming中

在server启动时，这行`listener = self.transport._listen(self.target)`代码,
创建topic 和topic.server的top Queue和 topic 的fanout Queue及相关Exchange和consumer

```python
# oslo\messaging\_drivers\amqpdriver.py

def listen(self, target):
    conn = self._get_connection(pooled=False)

    listener = AMQPListener(self, target, conn)

    # 创建kombu Queue，并对Queue进行消费， listener作为回调函数即调用
    
    # AMQPListener __call__方法
    
    conn.declare_topic_consumer(target.topic, listener)
    conn.declare_topic_consumer('%s.%s' % (target.topic, target.server),
                                listener)
    conn.declare_fanout_consumer(target.topic, listener)

    return listener
```
在AMQPListener的`__call__`中，将AMQPIncomingMessage放入self.incoming中

```python
def __call__(self, message):
    rpc_common._safe_log(LOG.debug, 'received %s', message)

    self.msg_id_cache.check_duplicate_message(message)
    ctxt = rpc_amqp.unpack_context(self.conf, message)

    self.incoming.append(AMQPIncomingMessage(self,
                                             ctxt.to_dict(),
                                             message,
                                             ctxt.msg_id,
                                             ctxt.reply_q))
```
### client端

RPCClient，发送消息。call调用时，发送前会使用msg_id创建一个Queue并监听
```python
transport = oslo.messaging.get_transport(CONF)
target = oslo.messaging.Target(topic="rpc", server="server")
RPCClient = oslo.messaging.RPCClient(transport, target)
RPCClient.call({}, "test")
```

```python
# oslo\messaging\rpc\client.py

result = self.transport._send(self.target, msg_ctxt, msg,
                              wait_for_reply=True, timeout=timeout)
```

```python
# oslo\messaging\_drivers\amqpdriver.py

def send(self, target, ctxt, message, wait_for_reply=None, timeout=None):
    return self._send(target, ctxt, message, wait_for_reply, timeout)
```
在_send方法中，`msg.update({'_reply_q': self._get_reply_q()})` 中初始化ReplyWaiter对象

```python
class ReplyWaiter(object):

    def __init__(self, conf, reply_q, conn, allowed_remote_exmods):
        self.conf = conf
        self.conn = conn
        self.reply_q = reply_q
        self.allowed_remote_exmods = allowed_remote_exmods

        self.conn_lock = threading.Lock()
        self.incoming = []
        self.msg_id_cache = rpc_amqp._MsgIdCache()
        self.waiters = ReplyWaiters()
        # 一个direct 队列和consumer，回调函数为 此类的__call__方法
        
        conn.declare_direct_consumer(reply_q, self)

    def __call__(self, message):
        # 返回数据存入self.incoming中
        
        self.incoming.append(message)
```
RPC远程调用后，等待返回结果, 并返回
```python
if wait_for_reply:
    result = self._waiter.wait(msg_id, timeout)
    if isinstance(result, Exception):
        raise result
    return result
```
```python
# oslo/messaging/_drivers/amqpdriver.py ReplyWaiter

def wait(self, msg_id, timeout):
    final_reply = None
    while True:
        if self.conn_lock.acquire(False):
            try:
                while True:
                    reply, ending = self._poll_connection(msg_id, timeout)
                    if reply:
                        final_reply = reply
                    elif ending:
                        return final_reply
            finally:
                self.conn_lock.release()
                self.waiters.wake_all(msg_id)
        else:
            reply, ending, trylock = self._poll_queue(msg_id, timeout)
            if trylock:
                continue
            if reply:
                final_reply = reply
            elif ending:
                return final_reply
```
reply_xxx 队列Exchange `auto-delete = True`，所以当finally中 `self.conn_lock.release()`，
连接关闭时，消息队列全部清除