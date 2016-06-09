---
title: 利用Queue实现的Flask下的资源池
layout: post
categories: web技术
---

最近在开发flask网站程序的时候，遇到一个资源竟态分配的问题，尝试使用python中的Queue来解决

# 1 业务场景

在我们的业务中，我们提供了很多台设备可以供用户链接使用，用户通过接口来申请设备使用，在数据库中维护了设备的状态，可用、忙碌和不可用，分配的策略是从数据库中取出可用的设备列表，然后通过调用一个测试链接函数来确认是否实际可用，如果确实可用则把该设备标记为忙碌，然后把设备信息返回给用户，用户根据信息链接该设备进行操作。程序使用Flask框架，结合sqlalchemy组件，第一版本代码结构大体如下：

```python
#从数据库获取所有的可用设备
idle_devices = Device.query.filter_by(state=DEVICE_STATE_IDLE).all()
#寻找第一个可用设备分配给用户
for device in idle_devices:
   if device_available(device):
       idle_device = device
       break
#没有可用设备返回空
if idle_device is None:
   return jsonify(BaseApi.api_success(""))

#更新设备表
idle_device.state = DEVICE_STATE_BUSY
db.session.add(idle_device)

return idle_device
```

# 2 问题

上面这个版本的代码在刚上线的时候就出问题了，设备表中只有一台可用设备时，却让两个用户都分配到了设备资源，也就是说设备被重复分配了，通过观察代码不难发现，在多用户访问的情况下，上面的代码是有问题的，从数据库中取到的可用设备，在真正分配的时候可能就不可用了。所以这个写法是有问题的。另外除了资源重复分配的问题，在每次用户申请的时候都要取出所有的可用设备挨个遍历可用，这种在设备数量很大的时候，是不可取的。尝试使用改进方案。

# 3 python Queue

Python中，队列是线程间最常用的交换数据的形式，Queue模块是提供队列操作的模块。

## 3.1 创建队列

```python
import Queue
q = Queue.Queue(maxsize = 10)
```

Queue.Queue类即是一个队列的同步实现。队列长度可为无限或者有限。可通过Queue的构造函数的可选参数maxsize来设定队列长度。如果maxsize小于1就表示队列长度无限。

## 3.2 插入值

```python
q.put(item)
```

调用队列对象的put()方法在队尾插入一个项目。put()有两个参数，第一个item为必需的，为插入项目的值；第二个block为可选参数，默认为1。如果队列当前为空且block为1，put()方法就使调用线程暂停,直到空出一个数据单元。如果block为0，put方法将引发Full异常。

## 3.3 从队列中取值

```python
q.get()
```

调用队列对象的get()方法从队头删除并返回一个项目。可选参数为block，默认为True。如果队列为空且block为True，get()就使调用线程暂停，直至有项目可用。如果队列为空且block为False，队列将引发Empty异常。

# 4 Flask的上下文环境

下表展示了Flask上下文环境的四种变量。

变量名      | 上下文        | 说明                
------------| --------------| -------------
current_app | 程序上下文    | 当前激活程序的程序实例
g           | 程序上下文    | 处理请求时用作临时存储的对象。每次请求都会重设这个变量
request     | 请求上下文    | 请求对象，封装了客户端发出的HTTP 请求中的内容
session     | 请求上下文    | 用户会话，用于存储请求之间需要“记住”的值的词典

在这里如果想做跨请求的数据共享，唯一适合使用的是current_app。

# 5 解决方案

结合Queue的特点和Flask的上下文环境，可以设计如下的解决方案，在应用启动的时候，从MYSQL中读出所有可用设备的id，添加到一个cuerrent_app范围的的Queue中，然后在每次设备分配的时候从队列中读取一个设备id，如果该id对应的设备不可用继续读取，直到读到可用设备或者队列为空，当每次用户释放设备的时候再将设备id添加到队列中。我在实际编码中使用Flask的请求钩子函数，定义了一个在所有请求到来之前运行的函数来初始化该队列代码如下：

```python
@api.before_app_first_request
def init_queue():
    idle_devices = Device.query.filter_by(state=DEVICE_STATE_IDLE).all()
    q = Queue.Queue(0)
    for device in idle_devices:
        q.put(device.id)
    app.devices = q
```

# 6 结果

经过这样的改进之后，没有再出现重复分配的问题了，同时也一定程度上提高了程序运行的效率。

当然这个程序有一个潜在的问题，就是只能支持单进程运行，如果同时起了多个进程还是会出现资源重复分配的问题，这里后续可以考虑使用Redis 队列来替代python的Queue模块。










