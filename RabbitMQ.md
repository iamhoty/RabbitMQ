# RabbitMQ

文档：https://www.cnblogs.com/alex3714/articles/5248247.html

b站：bilibili.com/video/BV1dt411e7H6?p=12&spm_id_from=pageDriver

## 1.安装

安装 http://www.rabbitmq.com/install-standalone-mac.html

安装python rabbitMQ module 

```shell
pip install pika
or
easy_install pika
or
源码
https://pypi.python.org/pypi/pika
```

## 2.介绍

实现最简单的队列通信

![img](https://images2015.cnblogs.com/blog/720333/201609/720333-20160923111427277-763273185.png)

消息中间件 - 消息队列

- 异步 
- 解耦
- 并发

支持多系统、多语言使用。

## 3. 基本使用

### 3.1 send端

```python
#!/usr/bin/env python
import pika 
credentials = pika.PlainCredentials('alex', 'alex3714')
connection = pika.BlockingConnection(pika.ConnectionParameters(
  'localhost', credentials=credentials))
# 建立通道
channel = connection.channel()
# 声明队列
channel.queue_declare(queue='hello')
# n RabbitMQ a message can never be sent directly to the queue, it always needs to go through an exchange.
channel.basic_publish(exchange='',
                      routing_key='hello', 
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

### 3.2 receive端

```python
# coding = utf8
import pika 
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
# You may ask why we declare the queue again ‒ we have already declared it in our previous code.``# We could avoid that if we were sure that the queue already exists. For example if send.py program``#was run before. But we're not yet sure which program to run first. In such cases it's a good``# practice to repeat declaring the queue in both programs.
channel.queue_declare(queue='hello') # 声明队列

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(
    callback, # 拿到消息执行的函数
    queue='hello', # 获取哪个队列的数据
    no_ack=True)
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming() # 开始消费 死循环一直等待
```

### 3.3 配置认证

**远程连接rabbitmq server的话，需要配置权限** 

首先在rabbitmq server上创建一个用户

```shell
sudo rabbitmqctl add_user alex alex3714
# 分配角色
rabbmitctl set_user_tags alex administrator
```

同时还要配置权限，允许从外面访问

```shell
sudo rabbitmqctl set_permissions -p / alex ".*" ".*" ".*"
```

set_permissions [-p *vhost*] {*user*} {*conf*} {*write*} {*read*}

- vhost

  The name of the virtual host to which to grant the user access, defaulting to /.

- user

  The name of the user to grant access to the specified virtual host.

- conf

  A regular expression matching resource names for which the user is granted configure permissions.

- write

  A regular expression matching resource names for which the user is granted write permissions.

- read

  A regular expression matching resource names for which the user is granted read permissions.  

客户端连接的时候需要配置认证参数，生产者需要添加

```python
credentials = pika.PlainCredentials('alex', 'alex3714')
connection = pika.BlockingConnection(pika.ConnectionParameters(
  '10.211.55.5',5672,'/',credentials=credentials))
channel = connection.channel()
```

查看消息队列的有多少数据

```shell
rabbitmqctl list_queues
# 重启
/etc/init.d/rabbitmq-server restart
```

## 4. 持久化

We have learned how to make sure that even if the consumer dies, the task isn't lost(by default, if wanna disable  use no_ack=True). But our tasks will still be lost if RabbitMQ server stops.

When RabbitMQ quits or crashes it will forget the queues and messages unless you tell it not to. Two things are required to make sure that messages aren't lost: we need to mark both the queue and messages as durable.

First, we need to make sure that RabbitMQ will never lose our queue. In order to do so, we need to declare it as *durable*:

```python
# 队列持久化
channel.queue_declare(queue='task_queue', durable=True)　　
```

Although this command is correct by itself, it won't work in our setup. That's because we've already defined a queue called hello which is not durable. RabbitMQ doesn't allow you to redefine an existing queue with different parameters and will return an error to any program that tries to do that. But there is a quick workaround - let's declare a queue with different name, for exampletask_queue:

```python
channel.queue_declare(queue='task_queue', durable=True)
```

This queue_declare change needs to be applied to both the producer and consumer code.

At that point we're sure that the task_queue queue won't be lost even if RabbitMQ restarts. Now we need to mark our messages as persistent - by supplying a delivery_mode property with a value 2.

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode= 2,# make message persistent 消息持久化
                      ))
```

消息提供者代码

```python
import pika
import time
connection= pika.BlockingConnection(pika.ConnectionParameters(
    'localhost'))
channel= connection.channel()
 
# 声明queue  队列持久化
channel.queue_declare(queue='task_queue', durable=True)
 
# n RabbitMQ a message can never be sent directly to the queue, it always needs to go through an exchange.
import sys
 
message= ' '.join(sys.argv[1:])or "Hello World! %s" % time.time()
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                          # 消息持久化 
                          delivery_mode=2, # make message persistent
                      )
                      )
print(" [x] Sent %r" % message)
connection.close()
```

消费者代码

```python
#_*_coding:utf-8_*_
 
import pika, time
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
    'localhost'))
channel= connection.channel()
 
channel.queue_declare(queue='task_queue', durable=True)

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(20)
    print(" [x] Done")
    print("method.delivery_tag",method.delivery_tag)
    # 如果防止消费者挂掉而丢失消息，添加如下配置：
    ch.basic_ack(delivery_tag=method.delivery_tag) # 随机生成的标识符
 
channel.basic_consume(callback,
                      queue='task_queue',
                      no_ack=False # 默认需要确认
                      )
 
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

只有消费者成功的消费并返回tag给生产者，队列的消息才会去除。如果消费者消费时挂掉，这条未被消费者返回tag的消息会发送给其他消费者进行消费，直到有消费者返回成功的tag。已经在消费的消息不会发送给其他消费者，除非这个消费者挂掉。

## 5. 消息公平分发

RabbitMQ会默认把p发的消息依次分发给各个消费者(c)，跟负载均衡差不多

先启动消息生产者，然后再分别启动3个消费者，通过生产者多发送几条消息，你会发现，这几条消息会被依次分配到各个消费者身上

如果Rabbit只管按顺序把消息发到各个消费者身上，不考虑消费者负载的话，很可能出现，一个机器配置不高的消费者那里堆积了很多消息处理不完，同时配置高的消费者却一直很轻松。为解决此问题，**可以在各个消费者端，配置perfetch=1**,意思就是告诉RabbitMQ在我这个消费者当前消息还没处理完的时候就不要再给我发新消息了。

![img](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

 

```python
channel.basic_qos(prefetch_count=1)
```

**带消息持久化+公平分发的完整代码**

生产者端

```python
#!/usr/bin/env python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.queue_declare(queue='task_queue', durable=True)
 
message= ' '.join(sys.argv[1:])or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode= 2,# make message persistent
                      ))
print(" [x] Sent %r" % message)
connection.close()
```

消费者端

```python
#!/usr/bin/env python
import pika
import time
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.queue_declare(queue='task_queue', durable=True)
print(' [*] Waiting for messages. To exit press CTRL+C')
 
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag= method.delivery_tag)
 
channel.basic_qos(prefetch_count=1)
channel.basic_consume(callback,
                      queue='task_queue')
 
channel.start_consuming(
```

## 6.Publish\Subscribe(消息发布\订阅)　

之前的例子都基本都是1对1的消息发送和接收，即消息只能发送到指定的queue里，但有些时候你想让你的消息被所有的Queue收到，类似广播的效果，这时候就要用到exchange了，只发送给**在线**的消费者，如果都不在线，消息广播完就消失

没有收听者一个队列

An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the *exchange type.*

Exchange在定义的时候是有类型的，以决定到底是哪些Queue符合条件，可以接收消息

fanout: 所有bind到此exchange的queue都可以接收消息（广播）
direct: 通过routingKey和exchange决定的那个唯一的queue可以接收消息（组播）
topic:所有符合routingKey(此时可以是一个表达式)的routingKey所bind的queue可以接收消息

```
表达式符号说明：
#代表一个或多个字符，会收到所有的管道的消息
*代表任何字符
例：#.a会匹配a.a，aa.a，aaa.a等
*.a会匹配a.a，b.a，c.a等
注：使用RoutingKey为#，Exchange Type为topic的时候相当于使用fanout　
```

headers: 通过headers 来决定把消息发给哪些queue

![img](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

### 6.1 广播 fanout

消息publisher

```python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='logs',
                         type='fanout') #  广播 不需要声明队列
 
message= ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs',
                      routing_key='', # 不需要指定队列 所有绑定到exchange的队列都会收到
                      body=message)
print(" [x] Sent %r" % message)
connection.close()
```

消息subscriber

```python
#_*_coding:utf-8_*_
__author__= 'Alex Li'
import pika
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='logs',
                         type='fanout')
# 不指定queue名字,rabbit会随机分配一个名字,exclusive=True会在使用此queue的消费者断开后,自动将queue删除
# 广播时 消费者需要有各自的队列 所以随机生成
result= channel.queue_declare(exclusive=True)
queue_name= result.method.queue

# 队列绑定到exchange上
channel.queue_bind(exchange='logs',
                   queue=queue_name)
 
print(' [*] Waiting for logs. To exit press CTRL+C')
 
def callback(ch, method, properties, body):
    print(" [x] %r" % body)
 
channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

# 开始消费
channel.start_consuming()
```

### 6.2 组播 direct

有选择的接收消息(exchange type=direct)　

RabbitMQ还支持根据关键字发送，即：队列绑定关键字，发送者将数据根据关键字发送到消息exchange，exchange根据 关键字 判定应该将数据发送至指定队列。

![img](http://www.rabbitmq.com/img/tutorials/python-four.png)

publisher

```python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='direct_logs',
                         type='direct')
 
severity= sys.argv[1] if len(sys.argv) >1 else 'info' # 级别 严重程度
message= ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity, # 哪些对列绑定了severity 哪些队列就能收到
                      body=message)
print(" [x] Sent %r:%r" % (severity, message))
connection.close()
```

subscriber　

```python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='direct_logs',
                         type='direct')
 
result= channel.queue_declare(exclusive=True)
queue_name= result.method.queue

# 绑定列表
severities= sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

# 绑定队列
for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)
 
print(' [*] Waiting for logs. To exit press CTRL+C')
 
def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))
 
channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)
 
channel.start_consuming()　　
```

### 6.3 topic

Although using the direct exchange improved our system, it still has limitations - it can't do routing based on multiple criteria.

In our logging system we might want to subscribe to not only logs based on severity, but also based on the source which emitted the log. You might know this concept from the [syslog](http://en.wikipedia.org/wiki/Syslog) unix tool, which routes logs based on both severity (info/warn/crit...) and facility (auth/cron/kern...).

That would give us a lot of flexibility - we may want to listen to just critical errors coming from 'cron' but also all logs from 'kern'.

![img](http://www.rabbitmq.com/img/tutorials/python-five.png)

publisher

```python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='topic_logs',
                         type='topic') # 改为topic
 
routing_key= sys.argv[1] if len(sys.argv) >1 else 'anonymous.info'
message= ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))
connection.close()
```

subscriber

```python
import pika
import sys
 
connection= pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel= connection.channel()
 
channel.exchange_declare(exchange='topic_logs',
                         type='topic')
 
result= channel.queue_declare(exclusive=True) # 排他
queue_name= result.method.queue
 
binding_keys= sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)
 
for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)
 
print(' [*] Waiting for logs. To exit press CTRL+C')
 
def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))
 
channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)
 
channel.start_consuming()
```

To receive all the logs run: 收所有

```
python receive_logs_topic.py "#"
```

To receive all logs from the facility "kern":

```
python receive_logs_topic.py "kern.*"
```

Or if you want to hear only about "critical" logs:

```
python receive_logs_topic.py "*.critical"
```

You can create multiple bindings:

```
python receive_logs_topic.py "kern.*" "*.critical"
```

And to emit a log with a routing key "kern.critical" type:

```
python emit_log_topic.py "kern.critical" "A critical kernel error"
```