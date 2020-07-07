---
layout: post
title: 'oslo.messaging RPC举例'
subtitle:   "example for oslo.messaging usage"
date:       2020-07-07 23:57:00
author:     "Caesar"
header-mask: 0.3
catalog:    true
tags:
    - openstack
---

## 配置文件
配置rabbit的ip和port，userid和password，如下是部署rabbitmq的主机
```
[DEFAULT]
rabbit_host=196.168.1.120
rabbit_port=5672
rabbit_use_ssl=false
rabbit_userid=guest
rabbit_password=guest
```
注意DEFAULT为大写，否则无法加载

## 服务端
服务端ip 196.168.1.120

```python
from oslo.config import cfg
import oslo.messaging
import sys
import time

argv = sys.argv
CONF = cfg.CONF
CONF(argv[1:])


class TestEndPoint():

    def test(self,ctxt):
        return "from server"

# 此时为endpoint实例化的对象

endpoints = [TestEndPoint(), ]

tansport = oslo.messaging.get_transport(CONF)
target = oslo.messaging.Target(topic="rpc", server="server")
server = oslo.messaging.get_rpc_server(tansport, target, endpoints, executor="blocking")

server.start()

while True:
    time.sleep(1)

server.stop()

server.wait()
```
## 客户端
客户端196.168.1.124
```python
from oslo.config import cfg
import oslo.messaging
import sys

argv = sys.argv
CONF=cfg.CONF
CONF(argv[1:])

transport = oslo.messaging.get_transport(CONF)
target = oslo.messaging.Target(topic="rpc", server="server")
RPCClient = oslo.messaging.RPCClient(transport, target)
result = RPCClient.call({}, "test")
print(result)
```
## 测试
在196.168.1.120上启动server端
`python rpc_server.py --config-file=rpc_config`
rabbitmq 创建topic，topic.server 2个topic类型队列，Exchange默认openstack，bingding-key和队列名称相同。

![rabbitmq_list_queues](\img\oslo_messaging_usage\\list_queues.png)

客户端发送数据
`python rpc_client.py --config-file=rpc_config`
发送成功，接收到服务端返回信息
![client_call_server](\img\oslo_messaging_usage\\client_call_server.png)