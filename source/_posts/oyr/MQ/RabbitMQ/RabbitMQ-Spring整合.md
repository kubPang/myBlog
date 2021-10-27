---
title: RabbitMQ-Spring整合
date: 2021-09-10 00:00:00
author: 神奇的荣荣
summary: ""
categories: ory-MQ
tags: 
    - MQ
    - 中间件
    - RabbitMQ
---

# Spring 整合

## 环境准备

- rabbitmq 版本：3.8.8
- spring 版本：5.1.7.RELEASE
- spring-amqp 版本：2.1.8.RELEASE

<!-- more -->

## 简单消息模式

### 生产者

#### 新建maven项目，导入依赖

```xml
<dependencies>
    <!-- spring上下文 & spring amqp -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.1.7.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit</artifactId>
        <version>2.1.8.RELEASE</version>
    </dependency>

    <!-- test -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>5.1.7.RELEASE</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.0</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### 编写配置文件

rabbitmq.properties（属性类）
```properties
# rabbitmq 基本配置
rabbitmq.host=106.52.180.14
rabbitmq.port=5672
rabbitmq.username=admin
rabbitmq.password=123
rabbitmq.virtual-host=/
```

spring-rabbitmq-producer.xml（spring配置类）
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

    <!--
        定义持久化队列，不存在则自动创建；不绑定到交换机则绑定到默认交换机
        默认交换机类型为direct，名字为：""，路由键为队列的名称
    -->
    <rabbit:queue id="spring_queue" name="spring_queue" auto-declare="true"/>

</beans>
```

#### 发送消息代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void hello() {
        // 给spring_queue发送消息，当前队列绑定到默认交换机，路由键为队列名
        rabbitTemplate.convertAndSend("spring_queue", "hello spring");
    }

}
```

### 消费者

#### 新建maven项目，导入依赖

```xml
<dependencies>
    <!-- spring上下文 & spring amqp -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.1.7.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit</artifactId>
        <version>2.1.8.RELEASE</version>
    </dependency>

    <!-- test -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>5.1.7.RELEASE</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.0</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### 编写配置文件

rabbitmq.properties（属性文件）
```properties
# rabbitmq 基本配置
rabbitmq.host=106.52.180.14
rabbitmq.port=5672
rabbitmq.username=admin
rabbitmq.password=123
rabbitmq.virtual-host=/
```

spring-rabbitmq-consumer.xml（spring配置文件）
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

    <!-- 监听器容器 -->
    <rabbit:listener-container connection-factory="connectionFactory">
        <!-- 监听器，指定监听的队列 -->
        <rabbit:listener ref="springQueueListener" queue-names="spring_queue"/>
    </rabbit:listener-container>

    <bean id="springQueueListener" class="com.oyr.rabbit.spring.listener.SpringQueueListener"/>
</beans>
```

#### 消费消息代码

```java
// 监听器对象
public class SpringQueueListener implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("消费消息：" + message);
    }

}
```

### 执行代码，查看结果

生产者发送消息，消费者成功消费消息。
当前模式是简单模式，声明一个队列，绑定到默认交换机上，由于是默认交换机（Direct类型），所以绑定的路由键是队列名，发送消息路由键需要和绑定路由键一致就可以成功消费消息。

## 订阅模式-Fanout

### 生产者

#### 修改配置

spring-rabbitmq-producer.xml（spring配置）
```xml
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

    <!-- ~~~~~~~~~~~~~~~~~~~~~~~~~~~~广播；所有绑定队列都能收到消息~~~~~~~~~~~~~~~~~~~~~~~~~~~~ -->
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_fanout_queue_1" name="spring_fanout_queue_1" auto-declare="true"/>
    <rabbit:queue id="spring_fanout_queue_2" name="spring_fanout_queue_2" auto-declare="true"/>

    <!--定义广播类型交换机；并绑定上述两个队列-->
    <rabbit:fanout-exchange id="spring_fanout_exchange" name="spring_fanout_exchange" auto-declare="true">
        <rabbit:bindings>
            <rabbit:binding queue="spring_fanout_queue_1"/>
            <rabbit:binding queue="spring_fanout_queue_2"/>
        </rabbit:bindings>
    </rabbit:fanout-exchange>
</beans>
```

#### 发送消息代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void fanoutTest() {
        // 给fanout交换机发送消息，所有绑定的队列都能接受到消息
        rabbitTemplate.convertAndSend("spring_fanout_exchange", "", "spring fanout");
    }
}
```

### 消费者

#### 修改配置

spring-rabbitmq-consumer.xml（spring配置）
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

    <!-- 监听器容器 -->
    <rabbit:listener-container connection-factory="connectionFactory">
        <!-- 监听器，指定监听的队列 -->
        <rabbit:listener ref="fanoutListener1" queue-names="spring_fanout_queue_1"/>
        <rabbit:listener ref="fanoutListener2" queue-names="spring_fanout_queue_2"/>
    </rabbit:listener-container>

    <!-- 监听器 -->
    <bean id="fanoutListener1" class="com.oyr.rabbit.spring.listener.FanoutListener1"/>
    <bean id="fanoutListener2" class="com.oyr.rabbit.spring.listener.FanoutListener2"/>

</beans>
```

#### 消息消费代码

FanoutListener1：
```java
public class FanoutListener1 implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("FanoutListener1消费消息：" + message);
    }

}
```

FanoutListener2：
```java
public class FanoutListener2 implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("FanoutListener2消费消息：" + message);
    }

}
```

### 执行代码，查看结果

生产者发送消息，消费者成功消费消息。
当前模式是订阅模式-Fanout，声明一个fanout类型的交换机，只要队列绑定到该交换机上，生产者发送消息后都可以消费到，fanout类型交换机忽略了路由键。

## 订阅模式-Topic

### 生产者

#### 修改配置

spring-rabbitmq-producer.xml（spring配置）
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

    <!-- ~~~~~~~~~~~~~~~~~~~~~~~~~~~~主题；*匹配一个单词，#匹配多个单词 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~ -->
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_topic_queue_star" name="spring_topic_queue_star" auto-declare="true"/>
    <rabbit:queue id="spring_topic_queue_well" name="spring_topic_queue_well" auto-declare="true"/>
    <rabbit:queue id="spring_topic_queue_well2" name="spring_topic_queue_well2" auto-declare="true"/>

    <rabbit:topic-exchange id="spring_topic_exchange" name="spring_topic_exchange" auto-declare="true">
        <rabbit:bindings>
            <rabbit:binding pattern="heima.*" queue="spring_topic_queue_star"/>
            <rabbit:binding pattern="heima.#" queue="spring_topic_queue_well"/>
            <rabbit:binding pattern="itcast.#" queue="spring_topic_queue_well2"/>
        </rabbit:bindings>
    </rabbit:topic-exchange>
</beans>
```

#### 发送消息代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring-rabbitmq-producer.xml")
public class ProducerTest {

    // 通过rabbitTemplate发送消息
    @Resource
    private RabbitTemplate rabbitTemplate;

    @Test
    public void topicTest() {
        // 给topic交换机发送消息，路由键进行模糊匹配
        rabbitTemplate.convertAndSend("spring_topic_exchange", "heima.oyr", "heima.oyr");
        rabbitTemplate.convertAndSend("spring_topic_exchange", "heima.laihong.nan", "heima.laihong.nan");
        rabbitTemplate.convertAndSend("spring_topic_exchange", "itcast.oyr", "itcast.oyr");
    }
}
```

### 消费者

#### 修改配置

spring-rabbitmq-consumer.xml（spring配置）
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

    <!-- 监听器容器 -->
    <rabbit:listener-container connection-factory="connectionFactory">
        <!-- 监听器，指定监听的队列 -->
        <rabbit:listener ref="topicListenerStar" queue-names="spring_topic_queue_star"/>
        <rabbit:listener ref="topicListenerWell" queue-names="spring_topic_queue_well"/>
        <rabbit:listener ref="topicListenerWell2" queue-names="spring_topic_queue_well2"/>
    </rabbit:listener-container>

    <!-- 监听器 -->
    <bean id="topicListenerStar" class="com.oyr.rabbit.spring.listener.TopicListenerStar"/>
    <bean id="topicListenerWell" class="com.oyr.rabbit.spring.listener.TopicListenerWell"/>
    <bean id="topicListenerWell2" class="com.oyr.rabbit.spring.listener.TopicListenerWell2"/>

</beans>
```

#### 消息消费代码

TopicListenerStar：
```java
public class TopicListenerStar implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("TopicListenerStar消费消息：" + message);
    }

}
```

TopicListenerWell：
```java
public class TopicListenerWell implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("TopicListenerWell消费消息：" + message);
    }

}
```

TopicListenerWell2：
```java
public class TopicListenerWell2 implements MessageListener {

    @Override
    public void onMessage(Message message) {
        System.out.println("TopicListenerWell2消费消息：" + message);
    }

}
```

### 执行代码，查看结果

生产者发送消息，消费者成功消费消息。  
spring_topic_queue_star队列有一条信息；spring_topic_queue_well队列有两条信息；spring_topic_queue_well2有一条消息；  
当前模式是订阅模式-Topic，声明一个topic类型的交换机，将队列绑定到该交换机上，绑定的路由键可以有约定模糊匹配功能，只需要跟消息路由键模糊匹配上就可以成功消费消息。

## 配置详解

### rabbitmq 链接相关

```xml
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
                            virtual-host="${rabbitmq.virtual-host}"/>
```

### rabbitAdmin

```xml
<!--
    定义RabbitAdmin（管理交换机、队列）
-->
<rabbit:admin connection-factory="connectionFactory"/>
```

### rabbitTemplate

```xml
    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>
```

### 声明队列

```xml
    <!--
        声明队列，不绑定到交换机则绑定到默认交换机
        id：bean id
        name：队列名称
        auto-declare：不存在自动创建
        auto-delete：是否自动删除，如果最后一个消费者连接断开是否自动删除
        durable：持久化
    -->
    <rabbit:queue id="spring_queue" name="spring_queue" durable="true" auto-delete="false" exclusive="false" auto-declare="true"/>
```

### 声明监听器

```xml
    <!--
        监听器容器
        connection-factory：连接工厂
     -->
    <rabbit:listener-container connection-factory="connectionFactory">
        <!--
            监听器，指定监听的队列与监听器的关系
            queue-names：监听的队列
            ref：监听对象的id
         -->
        <rabbit:listener queue-names="spring_topic_queue_star" ref="topicListenerStar"/>
    </rabbit:listener-container>
```

### 声明交换机

#### 声明Fanout交换机

```xml
    <!--
        声明fanout交换机
        id：bean id
        name：交换机名称
        auto-declare：是否自动创建
        durable：是否持久化
    -->
    <rabbit:fanout-exchange id="fanoutExchange" name="fanout_exchange" auto-declare="true" durable="true">
        <rabbit:bindings>
            <!--
                声明绑定关系（由于是广播模式，所以没有路由键）
                queue：队列名
             -->
            <rabbit:binding queue="spring_queue"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:fanout-exchange>
```

#### 声明Direct交换机

```xml
    <!--
        声明direct交换机
        id：bean id
        name：交换机名称
        auto-declare：是否自动创建
        durable：是否持久化
    -->
    <rabbit:direct-exchange id="directExchange" name="direct_exchange" auto-declare="true" durable="true">
        <rabbit:bindings>
            <!--
                声明绑定关系
                queue：队列名
                key：绑定路由键
             -->
            <rabbit:binding queue="spring_queue" key="spring_queue"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:direct-exchange>
```

#### 声明Topic交换机

```xml
    <!--
        声明topic交换机
        id：bean id
        name：交换机名称
        auto-declare：是否自动创建
        durable：是否持久化
    -->
    <rabbit:topic-exchange id="spring_topic_exchange" name="spring_topic_exchange" auto-declare="true">
        <rabbit:bindings>
            <!--
                声明绑定关系
                queue：队列名
                pattern：模糊路由键
             -->
            <rabbit:binding queue="spring_topic_queue_star" pattern="heima.*"/>
        </rabbit:bindings>
    </rabbit:topic-exchange>
```