---
title: RabbitMQ-高级
date: 2021-08-13 00:00:00
author: 神奇的荣荣
summary: ""
categories: ory-MQ
tags: 
    - MQ
    - 中间件
    - RabbitMQ
---

# 前提

当前文章代码演示，是基于amqp原生或spring整合的。

# 持久化

在生产过程中，难免会发生服务器宕机的事情，RabbitMQ也不例外，可能由于某种特殊情况下的异常而导致RabbitMQ宕机从而重启，那么这个时候对于消息队列里的数据，包括交换机、队列以及队列中存在消息恢复就显得尤为重要了。
RabbitMQ本身带有持久化机制，包括交换机、队列以及消息的持久化。持久化的主要机制就是将信息写入磁盘，当RabbtiMQ服务宕机重启后，从磁盘中读取存入的持久化信息，恢复数据。

## 交换机持久化

### 概念

交换机可以有两个状态：持久（durable）、暂存（transient）。
持久化的交换机会在消息代理（broker）重启后依旧存在，而暂存的交换机则不会（它们需要在代理再次上线后重新被声明）。

注意：并不是所有的应用场景都需要持久化的交换机。

### 交换机持久化实现

```java
// 声明交换机，并且指定持久化
boolean exchangeDurable = true;
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, exchangeDurable, false, null);
```

## 队列持久化

### 概念

持久化队列（Durable queues）会被存储在磁盘上，当消息代理（broker）重启的时候，它依旧存在。没有被持久化的队列称作暂存队列（Transient queues）。
持久化的队列并不会使得路由到它的消息也具有持久性。倘若消息代理挂掉了，重新启动，那么在重启的过程中持久化队列会被重新声明，无论怎样，只有经过持久化的消息才能被重新恢复。

注意：并不是所有的应用场景都需要持久化的队列。

### 队列持久化实现

```java
// 声明队列，并且指定持久化
boolean durable = true;
channel.queueDeclare(QUEUE_NAME, durable, false, false, null);
```

## 消息持久化

### 概念

消息能够以持久化的方式发布，RabbitMQ 会将此消息存储在磁盘上。如果服务器重启，持久化消息不会丢失。
将消息发送给一个持久化的交换机或者路由给一个持久化的队列，并不会使得此消息具有持久化性质：它完全取决与消息本身的持久模式（persistence mode）。将消息以持久化方式发布时，会对性能造成一定的影响（就像数据库操作一样，健壮性的存在必定造成一些性能牺牲）。

注意：RabbitMQ的消息是依附于队列存在的，所以想要消息持久化，那么前提是队列也要持久化。

### 消息持久化实现

要想让消息实现持久化需要在消息生产者修改代码，在发送消息时添加属性：MessageProperties.PERSISTENT_TEXT_PLAIN
```java
// 发送消息，并且指定持久化
// channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, "hello".getBytes());
```

# 消息确认（producer ack）

消息确认是针对消息生产者的。

## 概念

前提知识点：rabbitmq整个消息投递的路径为 producer -> rabbitmq broker -> exchange -> queue -> consumer。

在生产环境中由于一些不明原因，导致 rabbitmq 重启，在 RabbitMQ 重启期间生产者消息投递失败，导致消息丢失，需要手动处理和恢复。
思考：如何才能进行 RabbitMQ 的消息可靠投递呢？

> 消息确认包含两部分。  
> 第一部分：用来确认生产者是否成功将消息发送到 exchange。  
> 第二部分：用来确认 exchange 路由消息给 queue 的过程中，消息是否成功投递。

RabbitMQ 为我们提供了两种方式来确保消息的投递：
- confirm 确认模式
    - 对应消息确认第一部分
- return 退回模式
    - 对应消息确认第二部分

<!-- more -->

## confirm 确认模式

### 简介

消息从生产者到交换机会有一个 confirm 确认模式，当消息被 Borker 接收到就会触发 ConfirmCallback 进行回调，当前回调会包含消息是否成功发送到交换机中，生产者可以对针对失败情况做处理。

### 三种实现方式

Confirm的三种实现方式：
1. 普通发送确认模式
2. 批量确认模式
3. 异步监听确认模式

只有第三种方式是最长使用，这里只讲解第三种方式。

### 案例

#### 开启消息确认模式

connectionFactory.publisher-confirms="true"：开启确认模式
```xml
<rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                            port="${rabbitmq.port}"
                            username="${rabbitmq.username}"
                            password="${rabbitmq.password}"
                            virtual-host="${rabbitmq.virtual-host}" publisher-confirms="true"/>
```

#### 编写confirm回调

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void confirmTest() {
        // 消息发送到broker回调（返回ack，代表消息是否发送到交换机）
        RabbitTemplate.ConfirmCallback confirmCallback = (CorrelationData correlationData, boolean ack, String cause) -> {
            if (ack) {
                System.out.println("消息成功发送到交换机");
            } else {
                System.out.println("消息发送到交换机出现异常，异常消息：" + cause);
            }
        };

        // 设置确认回调
        rabbitTemplate.setConfirmCallback(confirmCallback);

        // 发送消息
        rabbitTemplate.convertAndSend("direct_exchange", "456", "boot hello");
    }
}
```

## return 退回模式

### 简介

消息从交换机到队列投递会有一个 retunr 退回模式，当消息没有被成功投递到队列中后会触发 ReturnCallback 进行回调，生产者可以针对这种情况做对应的处理。

### 案例

#### 开启消息退回模式

connectionFactory.publisher-returns="true"：开启退回模式
```xml
<rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                            port="${rabbitmq.port}"
                            username="${rabbitmq.username}"
                            password="${rabbitmq.password}"
                            virtual-host="${rabbitmq.virtual-host}"
                            publisher-returns="true"/>
```

#### 编写return回调

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void confirmTest() {
        // 消息如果不能路由，默认丢弃，设置通知给生产者
        rabbitTemplate.setMandatory(true);
        
        // 消息没有成功投递到队列回调
        RabbitTemplate.ReturnCallback returnCallback = (Message message, int replyCode, String replyText, String exchange, String routingKey) -> {
            System.out.println("message：" + message);
            System.out.println("exchange：" + exchange);
            System.out.println("routingKey：" + routingKey);
        };

        // 设置退回回调
        rabbitTemplate.setReturnCallback(returnCallback);

        // 发送消息
        rabbitTemplate.convertAndSend("direct_exchange", "1111", "boot hello");
    }
}
```

## 小结

### confirm 确认模式

- 设置connectionFactory.publisher-confirms="true"，开启确认模式
- 使用rabbitTemplate.setConfirmCallback设置回调函数，当消息发送到broker时触发回调方法，通过ack判断当前消息是否成功发送到交换机，如果失败，则需处理。

### return 回退模式

- connectionFactory.publisher-returns="true"，开启回退模式
- 使用rabbitTemplate.setReturnCallback设置回调函数，当消息从exchange路由到queue失败后，如果设置了rabbittemplate.setMandatory(true)参数，则消息退回给producer，并执行回调方法。

# 消息应答（consumer ack）

消息应答是针对消息消费者的。

## 概念

问题：消费者完成一个任务可能需要一段时间，如果其中一个消费者处理一个长的任务并仅只完成了部分突然它挂掉了，会发生什么情况。RabbitMQ 一旦向消费者传递了一条消息，便立即将该消息标记为删除。在这种情况下，突然有个消费者挂掉了，我们将丢失正在处理的消息。

为了保证消息在发送过程中不丢失，rabbitmq 引入消息应答机制，消息应答就是：**消费者在接收到消息并且处理该消息之后，告诉 rabbitmq 它已经处理了，rabbitmq 可以把该消息删除了。**

ack指Acknowledge，确认，表示消费者收到消息后的确认（应答）方式。  
rabbitmq ack有三种应答方式：
- 自动应答：ack="none"
- 手动应答：ack="manual"
- 根据情况自动应答：ack="auto"（根据逻辑判断是否抛出异常，自动进行ack或nack）

## 自动应答

消息发送后立即被认为已经传送成功，这种模式需要在高吞吐量和数据传输安全性方面做权衡，因为这种模式如果消息在接收到之前，消费者那边出现连接或者 channel 关闭，那么消息就丢失了,当然另一方面这种模式消费者那边可以传递过载的消息，没有对传递的消息数量进行限制，当然这样有可能使得消费者这边由于接收太多还来不及处理的消息，导致这些消息的积压，最终使得内存耗尽，最终这些消费者线程被操作系统杀死，所以这种模式仅适用在消费者可以高效并以某种速率能够处理这些消息的情况下使用。

### 自动应答演示

spring 整合下，消费者默认是自动应答。（listener-container.acknowledge默认是auto）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>
    <!--
        监听器容器
        connection-factory：连接工厂
     -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="auto">
        <!--
            监听器，指定监听的队列与监听器的关系
            queue-names：监听的队列
            ref：监听对象的id
         -->
        <rabbit:listener queue-names="ack_queue" ref="ackListener"/>
    </rabbit:listener-container>

    <!-- 监听器 -->
    <bean id="ackListener" class="com.oyr.rabbit.spring.listener.AckListener"/>
</beans>
```

监听器：(自动应答，无需手动应答)
```java
public class AckListener implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("监听器消费消息：" + message);
        System.out.println("body：" + new String(message.getBody()));
    }

}
```

### 自动应答存在的问题

修改消费者，添加异常，代码如下：
```java
public class AckListener implements MessageListener {

    @Override
    public void onMessage(Message message) {
        // 模拟异常
        int i = 1 / 0;
        System.out.println("监听器消费消息：" + message);
        System.out.println("body：" + new String(message.getBody()));
    }

}
```

启动消费者，消费消息：  
可以发现消息消费失败了，并且队列里面的消息消失了，而消费者还一直处于监听状态，一直在获取消息消费，但都消费不成功，这样就导致了消息的丢失。

## 手动应答

### 手动应答方法

* Channel.basicAck(long deliveryTag, boolean multiple)
    - 用于肯定确认，RabbitMQ 已知道该消息并且成功的处理消息，可以将其丢弃了
    - 参数deliveryTag：是消息唯一id；参数multiple：是否批量处理；
* Channel.basicNack(long deliveryTag, boolean multiple, boolean requeue)
    - 用于否定确认
    - 参数deliveryTag：是消息唯一id；参数multiple：是否批量处理；requeue：被拒绝的消息是否重新入队列；
* Channel.basicReject(long deliveryTag, boolean requeue)
    - 用于否定确认
    - 与basicNack相比少一个参数，区别在于basicNack可以拒绝多条消息，而basicReject一次只能拒绝一条消息

### Multiple 的解释

multiple 表示批量应答； 
```
void basicAck(long deliveryTag, boolean multiple) throws IOException;

void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

multiple 的 true 和 false 代表不同意思
* true
    - 代表批量应答 channel 上未应答的消息
    - 比如说 channel 上有传送 tag 的消息 5,6,7,8 当前 tag 是 8 那么此时5-8 的这些还未应答的消息都会被确认收到消息应答
* false
    - 只会应答 tag=8 的消息 5,6,7 这三个消息依然不会被确认收到消息应答

![批量应答](https://rong0624.gitee.io/images/MQ/RabbitMQ/1628672592616.jpg)

### 手动应答演示

#### 设置手动应答

listener-container.acknowledge="manual"：设置手动应答
```xml
<rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
    <!--
        监听器，指定监听的队列与监听器的关系
        queue-names：监听的队列
        ref：监听对象的id
        -->
    <rabbit:listener queue-names="ack_queue" ref="ackListener"/>
</rabbit:listener-container>
```

#### 监听器手动应答

```java
public class AckListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        try {
            System.out.println("监听器消费消息：" + message);
            System.out.println("body：" + new String(message.getBody()));
            // 手动ack应答
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            // 异常，拒绝消息
            // 参数3：true放回到mq中，false丢弃消息
            channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
        }
    }
}
```

## 小结

- ack有三种应答方式，none：自动应答，manual：手动应答，auto：根据情况确认
- 手动应答下，在消费者没有出现异常情况下，调用channel.basicAck进行手动应答。
- 手动应答下，在消费者出现异常情况下，调用channel.basicNack或channel.basicReject进行拒绝消息，根据情况判断消息是否重新入队。

# 消息预取

## 概念

- 本身消息的发送就是异步发送的，所以在任何时候，channel 上肯定不止只有一个消息，另外来自消费者的手动确认本质上也是异步的。
- 因此这里就存在一个未确认的消息缓冲区，因此希望开发人员能限制此缓冲区的大小，以避免缓冲区里面无限制的未确认消息问题。
- 这个时候就可以通过使用 basic.qos 方法设置“预取计数”值来完成的。该值定义通道上允许的未确认消息的最大数量。一旦数量达到配置的数量，RabbitMQ 将停止在通道上传递更多消息，除非至少有一个未处理的消息被确认，例如，假设在通道上有未确认的消息 5、6、7，8，并且通道的预取计数设置为 4，此时 RabbitMQ 将不会在该通道上再传递任何消息，除非至少有一个未应答的消息被 ack。比方说 tag=6 这个消息刚刚被确认 ACK，RabbitMQ 将会感知这个情况到并再发送一条消息。消息应答和 QoS 预取值对用户吞吐量有重大影响。通常，增加预取将提高向消费者传递消息的速度。虽然自动应答传输消息速率是最佳的，但是，在这种情况下已传递但尚未处理的消息的数量也会增加，从而增加了消费者的 RAM 消耗(随机存取存储器)应该小心使用具有无限预处理的自动确认模式或手动确认模式，消费者消费了大量的消息如果没有确认的话，会导致消费者连接节点的内存消耗变大，所以找到合适的预取值是一个反复试验的过程，不同的负载该值取值也不同 100 到 300 范围内的值通常可提供最佳的吞吐量，并且不会给消费者带来太大的风险。预取值为 1 是最保守的。
- 当然这将使吞吐量变得很低，特别是消费者连接延迟很严重的情况下，特别是在消费者连接等待时间较长的环境中。对于大多数应用来说，稍微高一点的值将是最佳的。

## 消息预取实现

```java
/**
 * 消息预取
 * 1. 确保ack为手动应答
 * 2. listener-container配置属性
 *      prefetch="1"
 */
public class AqsListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        // 1.获取消息
        System.out.println("监听器消费消息：" + message);

        // 2.处理业务逻辑

        // 3.ack消费者应答
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }

}
```

```xml
    <!--
        监听器容器
        connection-factory：连接工厂
        acknowledge="manual" 消息手动应答
        prefetch="1" 消费者每次从broker里面一次性取出几个待消费的消息，并且只有消息ack确认后，才会再去拉取。
    -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual" prefetch="1">
        <!--
            监听器，指定监听的队列与监听器的关系
            queue-names：监听的队列
            ref：监听对象的id
         -->
        <rabbit:listener queue-names="aqs_queue" ref="aqsListener"/>
    </rabbit:listener-container>
```

## 小结

- 消费者需要保证ack为手动应答模式（acknowledge="manual"）
- 消费者设置属性listener-container.prefetch，指定每次从broker里面一次性取出几个待消费的消息，并且只有消息ack确认后，才会再去拉取。

# TTL

TTL是什么呢？
> TTL是RabbitMQ中的一个消息或队列的属性，表明一条消息或该队列中的所有消息的最大存活时间，单位是毫秒。
> 换句话说，如果一条消息设置了TTL属性或者进入了设置TTL属性的队列，那么这条消息如果在TTL设置的时间内没有被消费，则会成为"死信".

有两种方式设置TTL
- 队列设置TTL
- 消息设置TTL

如果同时配置了队列的TTL和消息的TTL，哪个会生效呢？答案是较小的那个值会生效。

## 队列设置TTL

声明队列时，指定队列中消息的过期时间
```xml
    <!-- 定义队列 -->
    <rabbit:queue id="test.queue.ttl" name="test.queue.ttl">
        <!-- 设置队列的参数 -->
        <rabbit:queue-arguments>
            <!-- x-message-ttl 指定队列消息的过期时间 -->
            <entry key="x-message-ttl" value="10000" value-type="java.lang.Integer"/>
        </rabbit:queue-arguments>
    </rabbit:queue>
```

## 消息设置TTL

发送消息时，指定消息的过期时间
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void ttlHello() {
        MessagePostProcessor processor = new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                // 设置消息属性，消息的过期时间（单位：毫秒）
                message.getMessageProperties().setExpiration("5000");
                return message;
            }
        };
        rabbitTemplate.convertAndSend("test.exchange.ttl", "ttl.hello", "message ttl", processor);
    }
}
```

## 队列TTL与消息TTL区别

队列TTL 与 消息TTL是有区别的。
队列TTL：队列TTL，消息拥有统一的过期时间，一旦消息过期，就会成为死信。
消息TTL：消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间.

注意：还需要注意的一点是，如果不设置TTL，表示消息永远不会过期，如果将TTL设置为0，则表示除非此时可以直接投递该消息到消费者，否则该消息将会被丢弃。

## 小结

- 设置统一队列过期时间使用参数：x-message-ttl，单位：毫秒，会对整个队列消息设置统一的过期时间。
- 设置消息过期时间使用参数：expiration，单位：毫秒，可以对消息设置不同的过期时间。
- 队列设置过期时间和消息设置过期时间，如果两者都进行了设置，以时间短的为准。

# 死信队列

## 概念

先从概念解释上搞清楚这个定义，死信，顾名思义就是无法被消费的消息，字面意思可以这样理解，
一般来说，producer 将消息投递到 broker 或者直接到 queue 里了，consumer 从 queue 取出消息进行消费，
但某些时候由于特定的原因导致 queue 中的某些消息无法被消费，这样的消息如果没有后续的处理，就变成了死信，有死信自然就有了死信队列。

## 死信的来源

1. 消息TTL过期，消息到达超时时间未被消费
2. 队列达到最大长度（队列满了，无法再添加到mq中）
3. 消费者拒绝消息（basic.reject 或 basic.nack）并且requeue=false

## 死信队列实战

### 死信队列架构

![死信队列架构](https://rong0624.gitee.io/images/MQ/RabbitMQ/20211023213601.png)
![死信队列架构](https://rong0624.gitee.io/images/MQ/RabbitMQ/20211023221932.png)

死信队列架构图解：
> 第一步：声明正常的队列和正常的交换机
> 第二步：声明死信队列和死信交换机
> 第三步：正常队列绑定死信交换机并且指定消息成为死信的路由键
> 第四步：当前生产者发送消息后，消息由于某种原因成为死信，就会将死信发送给死信交换机，死信交换机会根据指定的死信路由键发送到相关的死信队列

### 搭建死信队列

修改生产者：spring-rabbitmq-producer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>
    <!--定义管理交换机、队列-->
    <rabbit:admin connection-factory="connectionFactory"/>

    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>

    <!-- 声明正常的队列（test.queue.dlx）和交换机（test.exchange.ttl） -->
    <rabbit:queue id="test.queue.dlx" name="test.queue.dlx">
        <rabbit:queue-arguments>
            <!-- 指定队列绑定死信交换机 -->
            <entry key="x-dead-letter-exchange" value="exchange.dlx" />
            <!-- 指定消息成为死信后的路由键 -->
            <entry key="x-dead-letter-routing-key" value="dlx.hello" />
            <!-- 指定队列最大长度 -->
            <entry key="x-max-length" value="10" value-type="java.lang.Integer" />
        </rabbit:queue-arguments>
    </rabbit:queue>
    <rabbit:topic-exchange name="test.exchange.dlx">
        <!-- 指定绑定关系 -->
        <rabbit:bindings>
            <rabbit:binding pattern="test.dlx.#" queue="test.queue.dlx"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

    <!-- 声明死信队列和交换机 -->
    <rabbit:queue id="queue.dlx" name="queue.dlx"/>
    <rabbit:topic-exchange name="exchange.dlx">
        <!-- 指定绑定关系 -->
        <rabbit:bindings>
            <rabbit:binding pattern="dlx.#" queue="queue.dlx"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

</beans>
```

### 消息TTL过期

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void dlxTest() {
        // 消息过期后，成为死信
        // 消息成为死信，会被发送到死信交换机并被路由到死信队列
        MessagePostProcessor processor = new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                // 设置消息属性，消息的过期时间（单位：毫秒）
                message.getMessageProperties().setExpiration("10000");
                return message;
            }
        };
        rabbitTemplate.convertAndSend("test.exchange.dlx", "test.dlx.hello", "message dlx", processor);
    }
}
```

### 队列达到最大长度

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void dlxTest() {
        // 队列最大长度为10，一共发送15个消息，有5个消息成功死信
        // 消息成为死信，会被发送到死信交换机并被路由到死信队列
        for (int i = 0; i < 15; i++) {
            rabbitTemplate.convertAndSend("test.exchange.dlx", "test.dlx.hello", "message dlx" + i);
        }
    }
}
```

### 消费者拒绝消息

消费者配置：spring-rabbitmq-consumer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>

    <!--
        监听器容器
        acknowledge="manual" 手动ack确认
    -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
        <!-- 监听器，指定监听的队列 -->
        <rabbit:listener ref="testDlxListener" queue-names="test.queue.dlx"/>
    </rabbit:listener-container>

    <bean id="testDlxListener" class="com.oyr.rabbitmq.consumer.DlxListener"/>
</beans>
```

```java
package com.oyr.rabbitmq.consumer;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.listener.api.ChannelAwareMessageListener;

public class DlxListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        System.out.println("监听器消费消息：" + message);
        System.out.println("body：" + new String(message.getBody()));
        // ack，拒绝消息，并且丢弃消息
        // 消费者拒绝消息并丢弃消息后，成为死信
        // 消息成为死信，会被发送到死信交换机并被路由到死信队列
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }

}
```

## 小结

- 死信交换机和死信队列和普通的没有区别（换了一个叫法）
- 当消息称为死信后，如果该队列绑定了死信交换机，则消息会被死信交换机路由到死信队列中
- 消息成为死信的情况
    - 消息TTL过期，消息到达超时时间未被消费
    - 队列达到最大长度（队列满了，无法再添加到mq中）
    - 消费者拒绝消息（basic.reject 或 basic.nack）并且requeue=false

# 延迟队列

## 概念

延时队列，首先，它是一种队列，队列意味着内部的元素是有序的，元素出队和入队是有方向性的，元素从一端进入，从另一端取出。

其次，延时队列，最重要的特性就体现在它的延时属性上，跟普通的队列不一样的是，普通队列中的元素总是等着希望被早点取出处理，而延时队列中的元素则是希望被在指定时间得到取出和处理，所以延时队列中的元素是都是带时间属性的，通常来说是需要被处理的消息或者任务。

简单来说，延时队列就是用来存放需要在指定时间被处理的元素的队列。

## 使用场景

1. 新用户注册成功后，7天后发送短信问候
2. 订单下单后，三十分钟未支付则自动取消订单
3. 用户发起退款，如果三天内没有被处理则通知相关的运营人员

## 实现延迟队列（TTL + 死信队列）

### 前提

队列TTL：队列TTL，消息拥有统一的过期时间，一旦消息过期，就会成为死信。
消息TTL：消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间

我们需要的效果是，消息拥有统一的过期时间，并且一旦消息过期，就会成为死信，所以使用队列TTL。

### 架构

![TTL+死信队列实现延迟队列](https://rong0624.gitee.io/images/MQ/RabbitMQ/20211023213601.png)
![TTL+死信队列实现延迟队列](https://rong0624.gitee.io/images/MQ/RabbitMQ/20211023221932.png)

### 生产者代码

spring-rabbitmq-producer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>
    <!--定义管理交换机、队列-->
    <rabbit:admin connection-factory="connectionFactory"/>

    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>

    <!-- 声明正常的队列（order.queue）和正常交换机（order.exchange） -->
    <rabbit:queue id="order.queue" name="order.queue">
        <rabbit:queue-arguments>
            <!-- 指定队列绑定死信交换机 -->
            <entry key="x-dead-letter-exchange" value="order.exchange.dlx" />
            <!-- 指定消息成为死信后的路由键 -->
            <entry key="x-dead-letter-routing-key" value="order.dlx.hello" />
            <!-- 指定队列消息的过期时间 -->
            <entry key="x-message-ttl" value="10000" value-type="java.lang.Integer"/>
        </rabbit:queue-arguments>
    </rabbit:queue>
    <rabbit:topic-exchange name="order.exchange">
        <!-- 指定绑定关系 -->
        <rabbit:bindings>
            <rabbit:binding pattern="order.#" queue="order.queue"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

    <!-- 声明死信队列和死信交换机 -->
    <rabbit:queue id="order.queue.dlx" name="order.queue.dlx"/>
    <rabbit:topic-exchange name="order.exchange.dlx">
        <!-- 指定绑定关系 -->
        <rabbit:bindings>
            <rabbit:binding pattern="order.dlx.#" queue="order.queue.dlx"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

</beans>
```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void delayTest() throws InterruptedException {
        // 队列TTL + 死信队列 = 延迟队列
        // 消息在10秒后成为死信
        // 消息成为死信后，被死信队列接受到
        rabbitTemplate.convertAndSend("order.exchange", "order.msg", "orderId:1,createTime:" + new Date());
        for (int i = 0; i < 10; i++) {
            System.out.println(i + 1);
            Thread.sleep(1000);
        }
    }
}
```

### 消费者代码

spring-rabbitmq-consumer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>

    <!--
        监听器容器
        acknowledge="manual" 手动ack确认
    -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
        <!-- 
            监听器，指定监听的队列
            注意，当前是监听死信队列
        -->
        <rabbit:listener ref="delayListener" queue-names="order.queue.dlx"/>
    </rabbit:listener-container>

    <bean id="delayListener" class="com.oyr.rabbitmq.consumer.DelayListener"/>
</beans>
```

```java
package com.oyr.rabbitmq.consumer;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.listener.api.ChannelAwareMessageListener;

public class DlxListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        System.out.println("监听器消费消息：" + message);
        System.out.println("body：" + new String(message.getBody()));
        // 消息在过期被死信队列接受到，最终被死信队列消费者消费
        // ack，拒绝消息，并且丢弃消息
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }

}
```

### 小结

rabbitMQ没有提供延迟队列功能，但是可以使用：TTL（队列TTL）+死信队列来实现延迟队列效果。

## 实现延迟队列（基于延迟插件）

### 前提

上文TTL+死信队列实现的延迟队列，队列消息过期时间是统一的。
现在想实现在消息粒度上添加TTL，并使其在设置的TTL时间及时死亡应该如何实现呢？

思考：消息TTL + 死信队列是否能实现？
> 答案是不能的。
> 消息TTL：消息即使过期，也不一定会被马上丢弃，因为消息是否过期是在即将投递到消费者之前判定的，如果当前队列有严重的消息积压情况，则已过期的消息也许还能存活较长时间。在这种情况下，过期的消息还可能继续存活，没有满足在指定时间要进行处理的情况。

如何解决？答案：mq提供的延迟插件。

### 架构

![基于延迟插件实现的延迟队列](https://rong0624.gitee.io/images/MQ/RabbitMQ/20211024170444.png)

图解：
> 生产者发送消息到延迟交换机中
> 延迟交换机在消息到达投递时间后，才将消息投递到对应的队列中

注意：在自定义的延迟交换机中，这是一种新的交换类型，该类型消息支持延迟投递机制 消息传递后并不会立即投递到目标队列中，而是存储在 mnesia(一个分布式数据系统)表中，当达到投递时间时，才投递到目标队列中。

### 延迟插件安装

1）下载rabbitmq_delayed_message_exchange插件
> https://www.rabbitmq.com/community-plugins.html 

2）将插件解压到RabbitMQ的插件目录
> cp ~/rabbitmq_delayed_message_exchange-3.8.0.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.8.8/plugins/

3）安装插件，并重启rabbitmq
> rabbitmq-plugins enable rabbitmq_delayed_message_exchange
> service rabbitmq-server restart

### 生产者代码

spring-rabbitmq-producer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>
    <!--定义管理交换机、队列-->
    <rabbit:admin connection-factory="connectionFactory"/>

    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>

</beans>
```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitAdmin管理组件
    @Resource
    private RabbitAdmin rabbitAdmin;
    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void delayTest() throws InterruptedException {
        // 声明交换机
        String exchangeName = "delay.exchange";
        String exchangeType = "x-delayed-message"; // 指定延迟交换机类型
        Map<String, Object> arg = new HashMap<>();
        arg.put("x-delayed-type" , "topic");
        Exchange delayExchange = new CustomExchange(exchangeName, exchangeType, true, false, arg);
        rabbitAdmin.declareExchange(delayExchange);

        // 声明队列
        String queueName = "delay.queue";
        Queue queue = new Queue(queueName, true, false, false);
        rabbitAdmin.declareQueue(queue);

        // 声明绑定关系
        Binding binding = BindingBuilder.bind(queue).to(delayExchange).with("delay.#").noargs();
        rabbitAdmin.declareBinding(binding);

        // 发送消息
        rabbitTemplate.convertAndSend(exchangeName, "delay.hello", "boy，发送一条20过期时间消息，现在时间" + new Date(), (msg) -> {
            msg.getMessageProperties().setDelay(20000);
            return msg;
        });
        rabbitTemplate.convertAndSend(exchangeName, "delay.hello", "girl，发送一条5秒过期时间消息，现在时间" + new Date(), (msg) -> {
            msg.getMessageProperties().setDelay(5000);
            return msg;
        });

        for (int i = 0; i < 20; i++) {
            System.out.println(i +1);
            Thread.sleep(1000);
        }
    }
}
```

### 消费者代码

spring-rabbitmq-consumer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>

    <!--
        监听器容器
        acknowledge="manual" 手动ack确认
    -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
        <!-- 监听器，指定监听的队列 -->
        <rabbit:listener ref="delayListener" queue-names="delay.queue"/>
    </rabbit:listener-container>

    <bean id="delayListener" class="com.oyr.rabbitmq.consumer.DelayListener"/>
</beans>
```

### 小结

在当前方式下，实现在消息粒度上的 TTL。
第一条消息时间为20秒，第二条消息时间为5秒。消费者成功按时间顺序消费到消息，这里不展示了。

## 小结

rabbitmq实现延迟队列有两种方式
- TTL（队列TTL） + 死信队列模式实现延迟队列
- 延迟插件实现延迟队列（消息时间在延迟交换机上判断）

# 优先级队列

## 使用场景

在我们系统中有一个订单催付的场景，我们的客户在天猫下的订单,淘宝会及时将订单推送给我们，如果在用户设定的时间内未付款那么就会给用户推送一条短信提醒，很简单的一个功能对吧，但是，tmall商家对我们来说，肯定是要分大客户和小客户的对吧，比如像苹果，小米这样大商家一年起码能给我们创
造很大的利润，所以理应当然，他们的订单必须得到优先处理，而曾经我们的后端系统是使用 redis 来存放的定时轮询，大家都知道 redis 只能用 List 做一个简简单单的消息队列，并不能实现一个优先级的场景，
所以订单量大了后采用 RabbitMQ 进行改造和优化,如果发现是大客户的订单给一个相对比较高的优先级，否则就是默认优先级。

## 概念

优先级队列，顾名思义，优先级高的具备优先消费的特权。

> 1.RabbitMQ在3.5.0版本的时候实现了优先级队列。任何一个队列都可以通过客户端配置参数方式设置一个优先级(但是不能使用策略的方式配置这个参数)。当前优先级的最大值为：255。这个值最好在1到10之间
> 2.队列的优先级设置只能通过声明方式设定，不能通过策略方式修改某个队列
> 3.消息发布者可以发送一个优先级消息通过basic.properties数字越大表示优先级越高
> 4.所谓的优先级队列就是，当消费者阻塞的时候，对具有优先级的消息直接按照优先级排序操作，然后按照优先级在一个一个的发送给消费者，这里需要多个条件(优先级队列、优先级消息、消费者阻塞、并且server对消费者排序)

## 优先级队列实现

### 生产者代码

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!--
        定义rabbitmq connectionFactory
        id：bean id
        host：mq服务地址
        port：mq服务端口
        username：用户名
        password：密码
     -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"
                               publisher-returns="true"/>
    <!--
        定义RabbitAdmin（管理交换机、队列）
    -->
    <rabbit:admin connection-factory="connectionFactory"/>

    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>

    <!-- 声明交换机 -->
    <rabbit:topic-exchange name="priority.exchange">
        <rabbit:bindings>
            <rabbit:binding pattern="priority.#" queue="priority.queue"></rabbit:binding>
            <rabbit:binding pattern="priority.#" queue="test.priority.queue"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

    <!-- 声明优先级队列 -->
    <rabbit:queue id="priority.queue" name="priority.queue">
        <rabbit:queue-arguments>
            <!-- 队列最大优先级设置为10 [当前优先级的最大值为：255。这个值最好在1到10之间] -->
            <entry key="x-max-priority" value="10" value-type="java.lang.Integer"/>
        </rabbit:queue-arguments>
    </rabbit:queue>

    <!-- 声明普通队列 -->
    <rabbit:queue id="test.priority.queue" name="test.priority.queue"/>
</beans>
```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void priorityTest() {
        for (int i = 0; i < 20; i++) {
            if (i % 3 == 0) {
                rabbitTemplate.convertAndSend("priority.exchange", "priority.hello", "优先级3，消息：" + i, (message) -> {
                    // 消息发送，指定优先级
                    message.getMessageProperties().setPriority(3);
                    return message;
                });
            } else if (i % 5 == 0) {
                rabbitTemplate.convertAndSend("priority.exchange", "priority.hello", "优先级5，消息" + i, (message) -> {
                    message.getMessageProperties().setPriority(5);
                    return message;
                });
            } else {
                rabbitTemplate.convertAndSend("priority.exchange", "priority.hello", "优先级1，消息" + i, (message) -> {
                    message.getMessageProperties().setPriority(1);
                    return message;
                });
            }
        }
    }
}
```

### 消费者代码

spring-rabbitmq-consumer.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--加载配置文件-->
    <context:property-placeholder location="classpath:rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>

    <!--
        监听器容器
        connection-factory：连接工厂
        acknowledge="manual" 消息手动应答
    -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
        <!-- 监听优先级队列 -->
        <rabbit:listener queue-names="priority.queue" ref="priorityListener"/>
        <!-- 监听普通队列 -->
        <rabbit:listener queue-names="test.priority.queue" ref="testPriorityListener"/>
    </rabbit:listener-container>

    <!-- 监听器 -->
    <bean id="priorityListener" class="com.oyr.rabbit.spring.listener.PriorityListener"/>
    <bean id="testPriorityListener" class="com.oyr.rabbit.spring.listener.TestPriorityListener"/>
</beans>
```

优先级队列监听器（优先级消息只有在优先级队列下才是按优先级排序的）
```java
public class PriorityListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        System.out.println("优先级队列消息：" + new String(message.getBody()));
        // 手动ack应答
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

普通队列监听器（优先级消息在普通队列下是正常顺序的，先进先出）
```java
public class TestPriorityListener implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        System.out.println("普通队列消息：" + new String(message.getBody()));
        // 手动ack应答
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

### 执行，查看结果

1）执行生产者，发送消息
> 发送消息后，普通队列和优先级队列都会有20条消息

2）启动消费者，监听消息
> 优先级队列监听器，会发现消息根据优先级排序消费 
> 普通队列监听器，会发现消息是正常顺序，按先进先出进行消费

3）启动消费者，执行生产者再次发送消息
> 发现，优先级监听器和普通队列监听器，消息消费顺序一致。
> 原因：优先级队列，其实就是就是在消息积压的时候进行按照优先级排序，如果是发一条消费一条就完全没有排序效果了。

## 小结

- 优先级队列，队列需要设置x-max-priority参数
- 优先级消息，消息需要设置priority参数
- 优先级队列必须和优先级消息一起使用，才能发挥出效果，但是会消耗性能
- 优先级队列必须在消费者繁忙的时候，才能对消息按照优先级排序
- 非优先级队列发送优先级消息是不会排序的，所以向非优先级队列发送优先级是没有任何作用的

# 惰性队列 

## 概念

RabbitMQ 从 3.6.0 版本开始引入了惰性队列的概念。
惰性队列会尽可能的将消息存入磁盘中，而在消费者消费到相应的消息时才会被加载到内存中，它的一个重要的设计目标是能够支持更长的队列，
即支持更多的消息存储。当消费者由于各种各样的原因(比如消费者下线、宕机亦或者是由于维护而关闭等)而致使长时间内不能消费消息造成堆积时，惰性队列就很有必要了。

默认情况下，当生产者将消息发送到RabbitMQ的时候，队列中的消息会尽可能的存储在内存之中，
这样可以更加快速的将消息发送给消费者。即使是持久化的消息，在被写入磁盘的同时也会在内存中驻留一份备份。
当RabbitMQ需要释放内存的时候，会将内存中的消息换页至磁盘中，这个操作会耗费较长的时间，也会阻塞队列的操作，进而无法接收新的消息。
虽然RabbitMQ的开发者们一直在升级相关的算法，但是效果始终不太理想，尤其是在消息量特别大的时候。

## 惰性队列实现

队列具备两种模式：default 和 lazy。默认的为 default 模式，在 3.6.0 之前的版本无需做任何变更。
lazy模式即为惰性队列的模式，可以通过调用 channel.queueDeclare 方法的时候在参数中设置，也可以通过Policy 的方式设置。
如果一个队列同时使用这两种方式设置的话，那么 Policy 的方式具备更高的优先级。
如果要通过声明的方式改变已有队列的模式的话，那么只能先删除队列，然后再重新声明一个新的。

```xml
<!-- 声明惰性队列 -->
<rabbit:queue id="lazy.queue" name="lazy.queue">
    <rabbit:queue-arguments>
        <!-- 设置当前队列模式为惰性队列 -->
        <entry key="x-queue-mode" value="lazy"/>
    </rabbit:queue-arguments>
</rabbit:queue>
```

## 小结

- 惰性队列，因为需要将消息存入磁盘中，减少了内存的开销，磁盘远远比内存大的，所以可以存储更多的消息。
- 惰性队列，因为需要将消息存入磁盘中，在消费时再取出，需要操作io，所以性能上不如普通队列。

# 备份交换机