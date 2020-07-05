---
layout: post
title: 'oslo.messaging Driver分析'
subtitle:   "oslo.messaging dirver analyse"
date:       2020-07-05 21:30:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - openstack
---
## oslo.messaging driver实现
(以rabbitmq实现为例)
oslo.messaging的dirver，使用ConnectionContext对ConnectionPool(连接池)和rabbitmq连接对象Connection进行管理。
### 对数据库connection管理
在impl_rabbitmq.py中，定义koubu操作rabbimq连接对象的Connection和RabbitDriver。
```python
class RabbitDriver(amqpdriver.AMQPDriverBase):

    def __init__(self, conf, url=None, default_exchange=None,
                 allowed_remote_exmods=[]):
        conf.register_opts(rabbit_opts)
        conf.register_opts(rpc_amqp.amqp_opts)
        # koubu操作的Connection
        
        # 与ConnectionContext继承的common.py中Connection不同
        
        connection_pool = rpc_amqp.get_connection_pool(conf, Connection)
        # 此处连接池中为Connection，调用其get方法，便会返回一个连接对象，
        
        # 配置rpc_conn_pool_size，定义其大小
        
        super(RabbitDriver, self).__init__(conf, connection_pool,
                                           url, default_exchange,
                                           allowed_remote_exmods)
```
在AMQPDriverBase中，定义_send, listen, get_reply_p, send_notification方法。以_send为例子
```python
def _send(self, target, ctxt, message,
          wait_for_reply=None, timeout=None, envelope=True):

    # FIXME(markmc): remove this temporary hack
    
    class Context(object):
        def __init__(self, d):
            self.d = d

        def to_dict(self):
            return self.d

    context = Context(ctxt)
    msg = message

    msg_id = uuid.uuid4().hex
    msg.update({'_msg_id': msg_id})
    LOG.debug('MSG_ID is %s' % (msg_id))
    rpc_amqp._add_unique_id(msg)
    rpc_amqp.pack_context(msg, context)

    msg.update({'_reply_q': self._get_reply_q()})

    if envelope:
        msg = rpc_common.serialize_msg(msg)

    if wait_for_reply:
        self._waiter.listen(msg_id)

    try:
        # ConnectionContext对象对Connection对象进行代理
        
        # 在__enter__方法中返回self
        
        with self._get_connection() as conn:
            if target.fanout:
            
                # 此处调用其get方法时，完成代理，发送fanout消息
                
                conn.fanout_send(target.topic, msg)
            else:
                topic = target.topic
                if target.server:
                    topic = '%s.%s' % (target.topic, target.server)
                    
                # 此处调用其get方法时，完成代理，发送消息到匹配topic
                
                conn.topic_send(topic, msg, timeout=timeout)
        if wait_for_reply:
            result = self._waiter.wait(msg_id, timeout)
            if isinstance(result, Exception):
                raise result
            return result
    finally:
        if wait_for_reply:
            self._waiter.unlisten(msg_id)
```
代码的实现，在其`__getattr__`中，
```python
def __getattr__(self, key):
    """Proxy all other calls to the Connection instance."""
    if self.connection:
    
        # self.connection数据库连接对象
        
        return getattr(self.connection, key)
    else:
        raise rpc_common.InvalidRPCConnectionReuse()
```
### 对资源池进行管理
如下所示，获取连接池，连接池中每个对象存入双向队列
```python
def get_connection_pool(conf, connection_cls):
    # 加锁
    
    with _pool_create_sem:
    
        # Make sure only one thread tries to create the connection pool.
        
        if not connection_cls.pool:
            connection_cls.pool = ConnectionPool(conf, connection_cls)
    return connection_cls.pool
```
数据库连接池继承下面的pool，实现create方法，如此每次需要时候，从资源池获取一个。
```python
@six.add_metaclass(abc.ABCMeta)
class Pool(object):

    """A thread-safe object pool.

    Modelled after the eventlet.pools.Pool interface, but designed to be safe
    when using native threads without the GIL.

    Resizing is not supported.
    """

    def __init__(self, max_size=4):
        super(Pool, self).__init__()

        self._max_size = max_size
        self._current_size = 0
        self._cond = threading.Condition()
        
        # 双向队列，默认右边插入
        
        self._items = collections.deque()

    def put(self, item):
        """Return an item to the pool."""
        
        # 放回资源池
        
        with self._cond:
            self._items.appendleft(item)
            self._cond.notify()

    def get(self):
        """Return an item from the pool, when one is available.

        This may cause the calling thread to block.
        """
        # 进入获取线程锁，退出释放锁
        
        with self._cond:
            while True:
                try:
                    return self._items.popleft()
                except IndexError:
                    pass

                if self._current_size < self._max_size:
                    self._current_size += 1
                    break

                # FIXME(markmc): timeout needed to allow keyboard interrupt
                # http://bugs.python.org/issue8844
                # 若无空闲资源，则等待
                
                self._cond.wait(timeout=1)

        # We've grabbed a slot and dropped the lock, now do the creation
        try:
            # 创建一个资源
            
            return self.create()
        except Exception:
            # 出错时，资源池数量恢复到创建前
            
            with self._cond:
                self._current_size -= 1
            raise
```
在ConnectionContex中的__exit__方法中定义 _done， _done方法是将对象调整后，put重新放回资源池。
```python

def _done(self):
    """If the connection came from a pool, clean it up and put it back.
    If it did not come from a pool, close it.
    """
    if self.connection:
        # 是否使用资源池存储
        
        if self.pooled:
            # 支持放回资源池
            
            # Reset the connection so it's ready for the next caller
            # to grab from the pool
            self.connection.reset()
            self.connection_pool.put(self.connection)
        else:
            try:
                # 不支持则关闭连接
                
                self.connection.close()
            except Exception:
                pass
        self.connection = None
```
查看Transport.py文件，在使用dirver提供的四种方法，建立新方法，这些方法完全可以创建起来server.py和Client对象。
