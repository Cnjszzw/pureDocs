# RocketMQ

## 同步发送消息：
producer发送消息到这个mq之后，需要等待mq发送成功或者失败的结果；

## 异步发送消息：
producer发送消息到mq之后，就不管了紧接着处理别的逻辑，不会阻塞着等待返回消失是否发送成功了，后续如果消息已经发送到mq了并且成功或者失败了，会再通过回调函数来告诉producer

## 单向发送关系：

不太关心发送到MQ之后，发送的结果，比如说发送日志的这个场景



## 消费者如何消费？

Push：broker向consumer主动push消息

Pull ：consumer 主动向这个broker拉消息

## 消费模式

1）广播模式

每一个consumer都去消费一遍消息，或者换种说法，每一个消息都被消费者去消费一遍，这个就是广播模式

2）负载均衡（默认情况下用这个负载均衡）

这个不同于广播模式，是多个消费者去共同消费一遍消息，一个消息只会被消费一遍，但是具体由哪个消费者进行消费，这个就和负载均衡算法有关系了

## 顺序消息

保证局部消息的顺序

## 延时消息

就是在producer发送的消息中设置延时的时间就可以了

## 批量消息

1000条消息，当然可以写一个for循环，循环1000次，发送这1000条，也可以放到一个集合中，一次性发送这1000条消息，这个就是批量消息

## 过滤消息

生产者发送消息到这个mq队列中，消费者可以有选择性的去进行过滤消费，比如根据这个tag进行过滤，然后也可以去根据自己自定义key-value值去消费

## 事务消息

去发送消息，到了mq，这个时候，这个消息只是一个半消息，对这个消费者不可见，需要等到producer所处的这个事务的执行情况，如果这个事务执行ok了，没问题，执行commit操作，这个消息才会可见，否则的话，如果那边执行事务失败，那么在事务过程中发送的这个消息也是不应该被消费的，会执行这个rollback操作，不会进一步让这个消费者去消费

##  四、消息的消费

消费者从Broker中获取消息的方式有两种：pull拉取方式和push推动方式。消费者组对于消息消费的模式又分为两种：集群消费Clustering和广播消费Broadcasting。

## 1. 推拉消费类型

## 拉取式消费

Consumer主动从Broker中拉取消息，主动权由Consumer控制。一旦获取了批量消息，就会启动消费过程。不过，该方式的实时性较弱，即Broker中有了新的消息时消费者并不能及时发现并消费。

## 推送式消费

该模式下Broker收到数据后会主动推送给Consumer。该消费模式一般实时性较高。

该消费类型是典型的发布-订阅模式，即Consumer向其关联的Queue注册了监听器，一旦发现有新的消息到来就会触发回调的执行，回调方法是Consumer去Queue中拉取消息。而这些都是基于Consumer与Broker间的长连接的。长连接的维护是需要消耗系统资源的。

## 对比

- **pull**: 需要应用去实现对关联Queue的遍历，实时性差；但便于应用控制消息的拉取
- **push**: 封装了对关联Queue的遍历，实时性强，但会占用较多的系统资源