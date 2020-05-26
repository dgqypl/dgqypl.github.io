---
layout: post
title:  "RabbitMQ简介"
author: mew151
image: assets/images/201904/nintendo-video-games-super-mario-mortal-kombat-cartridge-2048x1365-wallpaper.jpg
---
RabbitMQ是常用的开源消息中间件之一，本文会从最简单的模型开始，逐一介绍RabbitMQ的常用模型，并会说明在使用时的一些细节。（注：以下代码示例中使用Java语言）
#### 最简单的消息队列

![](/assets/images/201904/pc.png)

只包含一个队列，一个生产者（Producer），一个消费者（Consumer）。
在代码中，无论是Producer还是Consumer，都需要先初始化以下资源才可使用队列：
```java
ConnectionFactory factory = new ConnectionFactory();
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
```
connection：是socket connection的抽象，负责协议版本协商和鉴权等。
channel：实现绝大多数对RabbitMQ的操作（声明队列、publish消息、consume消息等）。
>The connection abstracts the socket connection, and takes care of protocol version negotiation and authentication and so on for us. A channel, which is where most of the API for getting things done resides.

##### Producer publish消息：
```java
channel.queueDeclare(QUEUE_NAME, ...);
channel.basicPublish(..., QUEUE_NAME, ..., message.getBytes("UTF-8"));
```
- 声明队列是幂等的，只有在队列不存在的时候才会创建它
- 对于一个已经存在的队列，不允许使用不同的参数来重声明
> Declaring a queue is idempotent - it will only be created if it doesn't exist already. RabbitMQ doesn’t allow you to redefine an existing queue with different parameters and will return an error to any program that tries to do that.

如果需要将队列和消息持久化，需要在声明队列的时候指定durable，在publish消息的时候指定props：
```java
channel.queueDeclare(QUEUE_NAME, durable: true, ...);
channel.basicPublish(..., QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
```
注意：即使设置了以上参数，也不能完全保证消息不丢失：
- 仍会存在很短的时间，RabbitMQ收到消息但并未将消息保存下来
- RabbitMQ不会按单条消息进行保存，而是先将消息缓存，再批量写入磁盘
>Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do `fsync(2)` for every message -- it may be just saved to cache and not really written to the disk.

如果需要进一步保证RabbitMQ成功收到消息，需要使用publisher confirms。（下文中有描述）
##### Consumer consume消息：

```java
channel.queueDeclare(QUEUE_NAME, durable: true, ...);
channel.basicConsume(QUEUE_NAME, ...);
```
注意：Consumer的声明队列和Producer的声明队列应保持一致。

默认情况下，需要Consumer来告知RabbitMQ收到消息了（即ack）：
```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8");
    try {
        ...
    } finally {
        channel.basicAck(...);
    }
};
channel.basicConsume(QUEUE_NAME, autoAck: false, deliverCallback, ...);
```
注意：如果设置成自动确认（即autoAck=true），那么RabbitMQ一旦将消息派送给Consumer，即认为消息被告知收到了。
>autoAck - true if the server should consider messages acknowledged once delivered; false if the server should expect explicit acknowledgements

#### 多Consumer的消息队列

![](/assets/images/201904/wq.webp)

消息队列中的消息可以分配给多个Consumer处理，默认RabbitMQ会按顺序依次将消息派送给这些Consumer，平均每个Consumer都会处理等量的消息，这种方式被叫做round-robin。
>By default, RabbitMQ will send each message to the next consumer, in sequence. On average every consumer will get the same number of messages. This way of distributing messages is called round-robin.

但在实际情况下，并不是每个Consumer压力都是相同的，有的处理的快，有的处理的慢，这时，对于那些未处理完消息的Consumer，RabbitMQ应该不再向其派送消息，而应该向其他空闲的Consumer派送。如果所有Consumer都处于busy状态，RabbitMQ会停止派送。

设置Qos的值，表示在channel中，允许最多未被ack的消息数量：
```java
channel.queueDeclare(...);
channel.basicQos(prefetchCount: 4);
channel.basicConsume(...);
```
一般来讲，对于提高系统吞吐量，Qos的值设置在100～300之间是比较合适的。（当然，具体问题还要具体分析，以上参考值只是官方推荐）
>The value defines the max number of unacknowledged deliveries that are permitted on a channel. Once the number reaches the configured count, RabbitMQ will stop delivering more messages on the channel unless at least one of the outstanding ones is acknowledged.

>Finding a suitable prefetch value is a matter of trial and error and will vary from workload to workload. Values in the 100 through 300 range usually offer optimal throughput and do not run significant risk of overwhelming consumers.

>Prefetch value of 1 is the most conservative. It will significantly reduce throughput, in particular in environments where consumer connection latency is high. For many applications, a higher value would be appropriate and optimal.

#### Exchange
**在RabbitMQ中，消息不会直接发送到队列中，而是会发送到exchange**，由exchange决定将消息发送到一个或多个队列中，或者将消息直接丢弃掉。

![](/assets/images/201904/exchanges.webp)

```java
Channel channel = connection.createChannel();
channel.exchangeDeclare(EXCHANGE_NAME, ...);
channel.basicPublish(...);
```
exchange一共有4种类型：`fanout`、`direct`、`topic`、`headers`
#### 发布／订阅模式
这种模式下，exchange的类型是`fanout`

![](/assets/images/201904/bindings.webp)

这样，当多个队列与同一个exchange绑定，当消息发送到exchange中，这些队列都会接收到这条消息。
##### Producer publish消息到exchange：
```java
Channel channel = connection.createChannel();
channel.exchangeDeclare(EXCHANGE_NAME, type: "fanout");
channel.basicPublish(EXCHANGE_NAME, routingKey: "", ...);
```
说明：fanout不需要指定routingKey（关于routingKey是什么，会在后面解释）
##### Consumer consume消息：
```java
Channel channel = connection.createChannel();
channel.exchangeDeclare(EXCHANGE_NAME, type: "fanout");
String queueName = channel.queueDeclare().getQueue(); // 自动生成队列的名字
channel.queueBind(queueName, EXCHANGE_NAME, routingKey: "");
channel.basicConsume(queueName, ...);
```
说明：自动生成的队列名字类似这样：amq.gen-JzTY20BRgKO-HjmUJj0wLg
#### 路由模式
这种模式下，exchange的类型是`direct`

![](/assets/images/201904/direct-exchange.webp)

在Producer publish消息时，需要指定routingKey；在Consumer将exchange和队列绑定时，需要指定binding key。当消息被发送到exchange中，exchange会寻找与消息的routingKey相等的binding key，如果有的话，并且binding key有绑定的队列，那么将消息派送到队列，否则丢弃消息。
>The routing algorithm behind a `direct` exchange is simple - a message goes to the queues whose `binding key` exactly matches the `routing key` of the message.

#### 多条件路由模式
这种模式下，exchange的类型是`topic`

![](/assets/images/201904/topic.webp)

topic exchange是direct exchange的升级版，routingKey不再是一个单词，而是一组由.分隔的单词，比如：quick.orange.rabbit

binding key也是类似，但可以使用通配符：
- *：代表1个单词
- \#：代表0个或多个单词

按照上图来举例：
- **quick.orange.rabbit**和**lazy.orange.elephant**会被发送到Q1和Q2
- **quick.orange.fox**会被发送到Q1
- **lazy.brown.fox**会被发送到Q2
- **lazy.pink.rabbit**只会被发送到Q2一次
- **quick.brown.fox**会被丢弃
- **orange**或**quick.orange.male.rabbit**也会被丢弃

思考：topic如何实现fanout和direct类型？
#### RPC
RabbitMQ是有RPC功能的，但一般大家都不会用，所以下面只简要介绍一下其结构：

![](/assets/images/201904/rpc.webp)

client发起请求调用server，以及server返回结果给client，是由上图中两个队列组成的，这里需要关注两个参数：

- correlationId：将request和response关联起来
- replyTo：server处理完之后将response发回给client的队列名

---

##### 关于consumer ack
ack一共有3种模式：
- `basic.ack`：positive的ack，指consumer收到消息并正常处理完成
- `basic.nack`：negative的ack，指consumer收到消息但没有处理
- `basic.reject`：与`basic.nack`含义一样，只不过不支持批量操作

>Manual acknowledgements can be batched to reduce network traffic. This is done by setting the `multiple` field of acknowledgement methods to `true`. Note that `basic.reject` doesn't historically have the field and that's why `basic.nack` was introduced by RabbitMQ as a protocol extension.

>It is possible to reject or requeue multiple messages at once using the `basic.nack` method. This is what differentiates it from `basic.reject`. It accepts an additional parameter, `multiple`.

```java
channel.basicNack(..., multiple: true, requeue: true);
channel.basicReject(..., requeue: true);
```
`basic.ack`和`basic.nack`/`basic.reject`对于RabbitMQ来讲，都会将消息从RabbitMQ中删除，区别最主要是语义上的。

当收到消息的Consumer处于busy状态，但其他Consumer有处于idle状态的，那么可以使用`basic.nack`/`basic.reject`来requeue该消息。

说明：当消息被requeue时，如果可能的话会被放到队列中它的原始位置，否则会被requeue到一个接近队列头的位置。
>When a message is requeued, it will be placed to its original position in its queue, if possible. If not (due to concurrent deliveries and acknowledgements from other consumers when multiple consumers share a queue), the message will be requeued to a position closer to queue head.

有一个标志位来标识一条消息是否被requeue了：
```java
boolean isRequeued = delivery.getEnvelope().isRedeliver();
```
>Redeliveries will have a special boolean property, `redeliver`, set to `true` by RabbitMQ. For first time deliveries it will be set to `false`. Note that a consumer can receive a message that was previously delivered to another consumer.

##### 关于publisher confirms
当Producer向RabbitMQ发送消息时，并不能保证消息被RabbitMQ成功接收到且处理了。为了保证RabbitMQ成功接收到消息，可以开启publisher confirms：
```java
Channel channel = connection.createChannel();
channel.confirmSelect();
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
         
    }
    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
         
    }
});
```
##### 关于RabbitMQ和Kafka的一些区别
RabbitMQ与Kafka一个很明显的不同之处在于，它的诸如队列声明、队列属性（队列是否可持久化等）都是由代码来控制的，而Kafka则是在broker侧配置。

---
#### 问答
###### 1、在RabbitMQ中，为什么有connection和channel的概念？
AMQP(Advanced Message Queuing Protocol)，高级消息队列协议，它可以使遵守这种协议的客户端能够与遵守这种协议的消息中间件进行通信。
>AMQP (Advanced Message Queuing Protocol) is a messaging protocol that enables conforming client applications to communicate with conforming messaging middleware brokers.

RabbitMQ是AMQP协议的一个实现。

AMQP是应用程序级别的协议，使用TCP长连接保证消息的可靠性。一般来讲，应用程序需要与RabbitMQ之间建立／维持多条连接，但同时建立多条TCP连接的代价是比较昂贵的。channel可以被认为是轻量级的connection，多个channel共享一条TCP连接。另外，对于客户端来讲，对AMQP协议的操作都是基于channel的。

如果应用程序是多线程的，那么最常见的做法是对于每个线程，为其创建一个channel。

更多细节参见：
- [AMQP 0-9-1 Model Explained — RabbitMQ#connections](https://www.rabbitmq.com/tutorials/amqp-concepts.html#amqp-connections)
- [AMQP 0-9-1 Model Explained — RabbitMQ#channels](https://www.rabbitmq.com/tutorials/amqp-concepts.html#amqp-channels)

###### 2、为什么在“最简单的消息队列”一节，consume消息时即使没将exchange和队列做绑定，也能正常收到Producer发送的消息？
RabbitMQ有一个默认的exchange，它是一个direct类型的exchange，名称是一个空字符串（""），它会自动绑定创建的每个队列，用队列名当作binding key。

更多细节参见：
- [AMQP 0-9-1 Model Explained — RabbitMQ#default exchange](https://www.rabbitmq.com/tutorials/amqp-concepts.html#exchange-default)

###### 3、在“publisher confirms”一节中，有提到RabbitMQ有可能会返给Producer nack的情况，那么什么时候会出现这种情况？
比如RabbitMQ队列长度达到限制了，以及其他的一些情况。

长度限制参见：
- [Queue Length Limit — RabbitMQ](https://www.rabbitmq.com/maxlength.html)

###### 4、Cosumer启用nack可能会导致死循环？
程序出现异常后，通过nack会把消息重新塞回队列头部，下一次又消费这条会出异常的消息，又出错，塞回队列......，导致后续消息堆积。因此使用nack的时候需要注意可能会出现这种情况。

相关参考：
- [RabbitMQ的ack或nack机制使用不当导致的队列堵塞或死循环问题](https://blog.csdn.net/youbl/article/details/80425959)

---
本文所有图片均来源于RabbitMQ官网

参考资料：
- [RabbitMQ tutorial - "Hello World!" — RabbitMQ](https://www.rabbitmq.com/tutorials/tutorial-one-java.htm)
- [RabbitMQ tutorial - Work Queues — RabbitMQ](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)
- [RabbitMQ tutorial - Routing — RabbitMQ](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)
- [Channel (RabbitMQ Java Client 5.6.0 API)](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/Channel.html)
- [Consumer Acknowledgements and Publisher Confirms — RabbitMQ](https://www.rabbitmq.com/confirms.html)
- [AMQP 0-9-1 Model Explained — RabbitMQ](https://www.rabbitmq.com/tutorials/amqp-concepts.html)