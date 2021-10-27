---
title: RabbitMQ-SpirngBoot整合
date: 2021-09-10 00:00:00
author: 神奇的荣荣
summary: ""
categories: ory-MQ
tags: 
    - MQ
    - 中间件
    - RabbitMQ
---

# SpringBoot 整合

## 环境准备

- rabbitmq 版本：3.8.8
- springboot 版本：2.2.4.RELEASE

<!-- more -->

## 订阅模式-Fanout

### 生产者

#### 新建maven项目，导入依赖

pom.xml
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

#### 编写配置类

配置类：（声明交换机，队列，绑定关系）
```
@Configuration
public class RabbitConfig {

    public final static String FANOUT_EXCHANGE = "boot.exchange.fanout";
    public final static String FANOUT_QUEUE = "boot.queue.fanout";

    // 1.声明交换机（交换机类型为fanout）
    @Bean
    public Exchange fanoutExchange() {
        return ExchangeBuilder.fanoutExchange(FANOUT_EXCHANGE).durable(true).build();
    }

    // 2.声明队列
    @Bean
    public Queue fanoutQueue() {
        return QueueBuilder.durable(FANOUT_QUEUE).build();
    }

    /**
     * 3.队列和交换机绑定
     * 需要知道哪个队列
     * 需要知道哪个交换机
     * 需要知道路由键（因为交换机类型是fanout，所以路由键为空）
     */
    @Bean
    public Binding fanoutBinding() {
        return BindingBuilder.bind(fanoutQueue()).to(fanoutExchange()).with("").noargs();
    }

}
```

#### 发送消息代码

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {Application.class})
public class ProducerTest {

    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void directTest() {
        /**
         * 发送消息
         * 参数1：指定交换机
         * 参数2：指定路由键（因为交换机类型是fanout，所以路由键为空，乱写也无所谓）
         * 参数3：消息
         */
        rabbitTemplate.convertAndSend(RabbitConfig.FANOUT_EXCHANGE, "123", "boot hello");
    }

}
```

### 消费者

#### 新建maven项目，导入依赖

pom.xml
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

#### 监听类（监听指定队列，消费消息）

```java
// 需要被spring容器扫描到
@Component
public class BootListener {

    // 监听指定的队列，进行消费消息
    @RabbitListener(queues = "boot.queue.fanout")
    public void listener(Message message) {
        System.out.println("message：" + message);
    }

}
```

### 执行，查看结果

这里不截图了，启动消费者进行监听，生产者发送消息后，消费者成功消费到消息。

### 小结

- 使用 springboot 整合 rabbitmq，将组件提供配置方式声明，简化编码
- 生产者使用 spring 提供的 rabbitTemplate 发送消息
- 消费者使用 spring 提供的 @RabbitListener 注解监听队列，消费消息

## 订阅模式-Direct（rabbitAdmin 与 注解方式）

### 生产者

#### 配置类

```java
@Configuration
public class RabbitConfig {

    public final static String DIRECT_EXCHANGE = "boot.exchange.direct";
    public final static String Direct_QUEUE = "boot.queue.direct";

    // 创建rabbitAdmin，并且通过rabbitAdmin创建交换机
    @Bean
    public RabbitAdmin rabbitAdmin(RabbitTemplate rabbitTemplate) {
        RabbitAdmin rabbitAdmin = new RabbitAdmin(rabbitTemplate);

        // 声明交换机
        Exchange exchange = new DirectExchange(DIRECT_EXCHANGE, true, false, null);
        rabbitAdmin.declareExchange(exchange);

        return rabbitAdmin;
    }

}
```

#### 生产者发送消息

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {Application.class})
public class ProducerTest {

    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void directTest() {
        /**
         * 发送消息
         * 参数1：指定交换机
         * 参数2：指定路由键（因为交换机类型是direct，消息路由键必须和绑定路由一致）
         * 参数3：消息
         */
        rabbitTemplate.convertAndSend(RabbitConfig.DIRECT_EXCHANGE, "123", "boot hello");
    }

}
```

### 消费者

#### 监听类（监听指定队列，消费消息）

```java
@Component
public class BootListener {

    /**
     * 监听指定的队列，进行消费消息
     * 通过 @RabbitListener 进行：声明队列，声明队列交换机的绑定关系，并且指定绑定key
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(name = "boot.queue.direct"),
            exchange = @Exchange(name = "boot.exchange.direct", type = ExchangeTypes.DIRECT),
            key = "123"))
    public void listener(@Payload String body, Message message) {
        System.out.println("body：" + body);
        System.out.println("message：" + message);
    }

}
```

### 执行，查看结果

这里不截图了，启动消费者进行监听，生产者发送消息后，消费者成功消费到消息。
因为当前是 direct 交换机，如果将消息的路由键改成456 ，那么消费者就不会有消息。

# @RabbitListener 使用

## 简介

在 SpringBoot 环境下，消费可以说比较简单了，借助@RabbitListener注解，基本上可以满足你 90% 以上的业务开发需求。
下面我们来看一下@RabbitListener的最最常用使用姿势。

前提：
对于消费者而言其实是不需要管理 exchange 的创建/销毁的，它是由发送者定义的；
一般来讲，消费者更关注的是自己的 queue，包括定义 queue 并与 exchange 绑定，而这一套过程是可以直接通过 rabbitmq 的控制台操作的哦。

## queue，exchange, binding 已存在

所以实际开发过程中，exchange 和 queue 以及对应的绑定关系已经存在的可能性是很高的，并不需要再代码中额外处理；  
在这种场景下，消费数据，可以说非常非常简单了，如下：
```java
@Component
public class BootListener {

    /**
     * 队列，队列与交换机绑定关系已经存在时，直接指定队列名进行消费
     */
    @RabbitListener(queues = "boot.queue.direct")
    public void listener(Message message) {
        System.out.println("body：" + new String(message.getBody()));
        System.out.println("message：" + message);
    }

}
```

## queue 不存在

不存在 Queue 的情况下，就需要我们来主动创建 Queue，并建立与 Exchange 的绑定关系，下面给出@RabbitListener的推荐使用姿势：
```java
@Component
public class BootListener {

    /**
     * 队列不存在时，需要创建一个队列，并且与exchange绑定
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(name = "boot.queue.direct"),
            exchange = @Exchange(name = "boot.exchange.direct", type = ExchangeTypes.DIRECT),
            key = "123"))
    public void listener(Message message) {
        System.out.println("body：" + new String(message.getBody()));
        System.out.println("message：" + message);
    }

}
```

一个注解，内部声明了队列，并建立绑定关系，就是这么神奇！！！以上，就是在队列不存在时的使用姿势，看起来也不复杂。

注意@QueueBinding注解的三个属性：
- value: @Queue 注解，用于声明队列，value 为 queueName, durable 表示队列是否持久化, autoDelete 表示没有消费者之后队列是否自动删除
- exchange: @Exchange 注解，用于声明 exchange， type 指定消息投递策略，我们这里用的 direct 方式
- key: 在 direct 方式下，这个就是我们熟知的 routingKey

## ack（消息应答）

rabbitmq 的核心知识点学习过程中，我们知道有一个消息应答机制，主要是针对消费者的。
默认情况下，消息应答机制是自动ack，我们可以通过 ack 方式(noack, auto, manual)，进行修改，可以如下处理：

```java
    /**
     * 设置为手动ack应答，在代码中显式进行ack应答
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(name = "boot.queue.direct"),
            exchange = @Exchange(name = "boot.exchange.direct", type = ExchangeTypes.DIRECT),
            key = "123"), ackMode = "MANUAL")
    public void listener(Message message, Channel channel) throws Exception {
        System.out.println("body：" + new String(message.getBody()));
        System.out.println("message：" + message);
        // 手动ack应答
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
```

请注意，这里使用了Channel和DeliveryTag：
- Channel：mq 和 consumer 之间的信道，通过它来 ack/nak
- DeliveryTag：消息的唯一标识，用于 mq 辨别是哪个消息被 ack/nak 了

当我们正确消费时，通过调用 ```basicAck``` 方法即可
```java
// 用于肯定确认，RabbitMQ 已知道该消息并且成功的处理消息，可以将其丢弃了
channel.basicAck(deliveryTag, false);
```

当我们消费失败，需要将消息重新塞入队列，等待重新消费时，可以使用 ```basicNack```
```java
// 用于否定确认，第三个参数true，表示这个消息会重新进入队列
channel.basicNack(deliveryTag, false, true);
```

当我们消费失败，想直接丢弃这个消息，可以使用 ```basicReject```
```java
// 用于否定确认，直接丢弃当前消息，不再入队
channel.basicReject(message.getMessageProperties().getDeliveryTag(), false);
```

## 并发消费

当消息很多，一个消费者吭哧吭哧的消费太慢，但是我的机器性能又杠杠的，这个时候我就希望并行消费，相当于同时有多个消费者来处理数据.

要支持并行消费，如下设置即可：
```java
@RabbitListener(bindings = @QueueBinding(value = @Queue(name = "boot.queue.direct"),
        exchange = @Exchange(name = "boot.exchange.direct", type = ExchangeTypes.DIRECT),
        key = "123"), concurrency = "4")
public void listener(Message message, Channel channel) throws Exception {
    System.out.println("body：" + new String(message.getBody()));
    System.out.println("message：" + message);
}
```

请注意注解中的concurrency = "4"属性，表示固定 4 个消费者；

## @Payload 与 @Headers

使用 @Payload 和 @Headers 注解可以消息中的 body 与 headers 信息
```java
@RabbitListener(queues = "debug")
public void listener(@Payload String body, @Headers Map<String, Object> headers) {
    System.out.println("body：" + body);
    System.out.println("headers：" + headers);
}
```

也可以获取单个 Header 属性
```java
@RabbitListener(queues = "debug")
public void listener(@Payload String body, @Headers String token) {
    System.out.println("body：" + body);
    System.out.println("token：" + token);
}
```

## @RabbitListener 和 @RabbitHandler 搭配使用

@RabbitListener 可以标注在类上面，需配合 @RabbitHandler 注解一起使用  
@RabbitListener 标注在类上面表示当有收到消息的时候，就交给 @RabbitHandler 的方法处理，具体使用哪个方法处理，根据 MessageConverter 转换后的参数类型

```
@Component
@RabbitListener(queues = "consumer_queue")
public class Receiver {

    @RabbitHandler
    public void processMessage1(String message) {
        System.out.println(message);
    }

    @RabbitHandler
    public void processMessage2(byte[] message) {
        System.out.println(new String(message));
    }
    
}
```

# RabbitAdmin 使用

## 简介

RabbitAdmin主要用于在Java代码中对理队和队列进行管理，用于创建、绑定、删除队列与交换机，发送消息等。

## 创建RabbitAdmin

查看源码发现，要创建RabbitAdmin，需要传递一个 ConnectionFactory 或 RabbitTemplate 即可。
```java
	/**
	 * Construct an instance using the provided {@link ConnectionFactory}.
	 * @param connectionFactory the connection factory - must not be null.
	 */
	public RabbitAdmin(ConnectionFactory connectionFactory) {
		Assert.notNull(connectionFactory, "ConnectionFactory must not be null");
		this.connectionFactory = connectionFactory;
		this.rabbitTemplate = new RabbitTemplate(connectionFactory);
	}

	/**
	 * Construct an instance using the provided {@link RabbitTemplate}. Use this
	 * constructor when, for example, you want the admin operations to be performed within
	 * the scope of the provided template's {@code invoke()} method.
	 * @param rabbitTemplate the template - must not be null and must have a connection
	 * factory.
	 * @since 2.0
	 */
	public RabbitAdmin(RabbitTemplate rabbitTemplate) {
		Assert.notNull(rabbitTemplate, "RabbitTemplate must not be null");
		Assert.notNull(rabbitTemplate.getConnectionFactory(), "RabbitTemplate's ConnectionFactory must not be null");
		this.connectionFactory = rabbitTemplate.getConnectionFactory();
		this.rabbitTemplate = rabbitTemplate;
	}
```

## RabbitAdmin创建交换机

```java
// 创建交换机，无则创建有则跳过，存在交换机但参数不一致则报错
rabbitAdmin.declareExchange(new FanoutExchange("test.exchange.fanout", true, false));
rabbitAdmin.declareExchange(new DirectExchange("test.exchange.direct", true, false));
rabbitAdmin.declareExchange(new TopicExchange("test.exchange.topic", true, false));
```

## RabbitAdmin创建队列

```java
// 创建队列
rabbitAdmin.declareQueue(new Queue("test.queue"));
```

## RabbitAdmin绑定交换机与队列

```java
//绑定队列
rabbitAdmin.declareBinding(new Binding("test.queue", Binding.DestinationType.QUEUE, "test.exchange.topic", "#", new HashMap<String, Object>()));        
```

## RabbitAdmin发送消息

```java
//发送消息
rabbitAdmin.getRabbitTemplate().convertAndSend("test.exchange.topic", "hello", "abc123");
```