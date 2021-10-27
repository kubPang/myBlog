---
title: RabbitMQ-基础
date: 2021-07-13 00:00:00
author: 神奇的荣荣
summary: ""
categories: ory-MQ
tags: 
    - MQ
    - 中间件
    - RabbitMQ
---

# RabbitMQ 基本概念

## 简介

RabbitMQ是由erlang语言开发，基于AMQP（Advanced Message Queue 高级消息队列协议）协议实现的消息中间件。  

RabbitMQ 是一个消息中间件：它接受并转发消息。你可以把它当做一个快递站点，当你要发送一个包裹时，
你把你的包裹放到快递站，快递员最终会把你的快递送到收件人那里，按照这种逻辑 RabbitMQ 是一个快递站，
一个快递员帮你传递快件。RabbitMQ 与快递站的主要区别在于，它不处理快件，而是接收，存储和转发消息数据。

RabbitMQ官方地址：http://www.rabbitmq.com

<!-- more -->

## 核心概念

![AMQP模式](https://rong0624.gitee.io/images/MQ/amqp模型.png)

RabbitMQ核心概念：
- Broker
    - 表示消息队列服务器实体（一个进程）。  
    - 一个server，接受客户端的连接，上线AMQP实体服务。
- Connection
    - 连接.
    - 应用程序与broker的网络连接，TCP/IP套接字连接。
- Channel 
    - 消息通道 
    - 几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道，客户端可以建立对多个Channel，每个Channel代表一个会话任务。
- Exchange
    - 交换机，用来接受生产者发送的消息，并将这些消息路由转发到某个队列。
- Queue
    - 消息队列，存储消息，用于发送给消费者。
    - 它是消息的容器，也是消息的终点。一个消息可以投入多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。
- Binding
    - 绑定，消息队列和交换器之间的关联。
    - 一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。
- Routing Key
    - 路由关键字，一个消息头，交换机可以用这个消息头决定如何路由某条消息。
- Message
    - 消息
    - 消息是不具名的，它由消息头和消息体组成。消息是不透明的，而消息头则由一系列的可选属性组成，这些属性包括routing-key（路由键）> ，priority（相对于其他消优先权），delivery-mode（指出该消息可能需要持久性存储）等
- Publisher
    - 消息生产者，是一个向交换器发布消息的客户端应用程序（进程）。
- Consumer
    - 消息消费者，是一个从消息队列中取得消息的客户端应用程序（进程）。
- Virtual Host  
    - 虚拟主机

## 消息模式

![消息模式](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626769058993.jpg)

1. 简单模式（simple）
2. 工作队列模式（work queues）
3. 订阅模式-Fanout（publish/subscribe）
4. 订阅模式-Direct（routing）
5. 订阅模式-Topic（topics）

**注意：订阅模式-Fanout，订阅模式-Direct，订阅模式-Topic都属于发布/订阅模式类型。**

# RabbitMQ 安装和配置

## 相关版本

```
erlang 21.3.x
rabbitmq 3.8.8
```

Erlang rpm下载：  
https://github.com/rabbitmq/erlang-rpm/releases

Rabbitmq rpm下载：  
https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.8/rabbitmq-server-3.8.8-1.el7.noarch.rpm

## Linux下安装

### 环境准备

```
linux（CentOS 7.5）
erlang 21.3.x
rabbitmq 3.8.8
```

### 安装Erlang

![erlang版本信息](https://rong0624.gitee.io/images/MQ/RabbitMQ/erlang版本.png)   
下载erlang时需要注意，要和rabbitmq版本兼容.

1）erlang rpm下载：
https://github.com/rabbitmq/erlang-rpm/releases/download/v21.3.1/erlang-21.3.1-1.el7.x86_64.rpm
erlang-21.3.1-1.el7.x86_64.rpm

2）rpm上传到系统中，安装erlang 
rpm -ivh erlang-21.3-1.el7.x86_64.rpm

3）查看erlang版本
erl -v

### 安装socat

安装Erlang后直接安装RabbitMQ，需要安装socat。

安装socat：  
yum install socat -y

### 安装RabbitMQ

```
1）rabbitmq rpm下载  
https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.8/rabbitmq-server-3.8.8-1.el7.noarch.rpm

2）rpm上传到系统中，并安装rabbitmq  
rpm -ivh rabbitmq-server-3.8.8-1.el7.noarch.rpm 

3）启动服务并测试  
# 启动服务 
service rabbitmq-server start

# 查看服务状态
service rabbitmq-server status

4）常用命令

# 启动服务 
service rabbitmq-server start 

# 查看服务状态
service rabbitmq-server status

# 停止服务 
service rabbitmq-server stop

# 重启服务 
service rabbitmq-server restart

# 开机自动启动 
chkconfig rabbitmq-server on
```

## Windos下安装

## Mac下安装

## RabbitMQ 管理界面及授权操作

### 管理界面

1）默认情况下，是没有安装web端的客户端插件，需要安装才可以生效。
```shell
rabbitmq-plugins enable rabbitmq_management
```
注意：管理界面会在15672端口提供服务

2）安装完毕以后，重启服务即可
```shell
service rabbitmq-server restart
```

3）在浏览器访问  

```
# 关闭防火墙服务
## 关闭防火墙
systemctl stop firewalld
## 关闭防火墙开机启动
systemctl disable firewalld
# 注意：一定要记住，在对应服务器（阿里云，腾讯云等）的安全组中开放15672端口

# 访问web管理界面
http://106.52.180.14:15672
```

成功访问：![RabbitMQ管理界面](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626777167279.jpg)

### 授权账号和密码

![guest登录](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721213622.png)  
说明：rabbitmq有一个默认账号和密码是：guest/guest，但guest默认情况只能在localhost本机下访问，所以需要添加一个远程登录的用户。

1）新增用户并授权：
```
#新增用户
rabbitmqctl add_user admin 123

#设置用户分配操作权限
rabbitmqctl set_user_tags admin administrator

#为用户添加资源权限
#set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
#解释：用户 admin 具有 / 这个 virtual host 中所有资源的配置、写、读权限
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
```

2）使用admin登录管理页面
登录成功：  
![admin登录](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721215337.png)  

### 小结

管理用户常见命令如下：
```
# 查看用户列表
rabbitmqctl list_users

# 新增账号[并设置密码]
rabbitmqctl add_user 账号 密码

# 修改密码
rabbitmqctl change_password 账号 新密码

# 删除账号
rabbitmqctl delete_user 账号

# 给账号设置角色
rabbitmqctl set_user_tags 账号 角色

# 给账号设置权限
rabbitmqctl set_permissions -p "/" 账号 ".*" ".*" ".*"
```

# 快速入门（simple）

## 环境准备

在学习RabbitMQ前必须掌握以下内容：  
熟悉使用Java  
熟悉使用Maven进行项目构建和依赖管理  
熟练使用eclipse或Idea开发工具  

环境约束：  
Jdk8  
Maven3.x  
Idea2019  
RabbitMQ 3.8.8

## 实现需求

需求：使用简单模式完成消息传递;

1. 创建生产者程序，发送消息
2. 创建消费者程序，消费消息

## 导入依赖

```xml
    <!-- rabbitmq 依赖客户端 -->
    <dependency>
        <groupId>com.rabbitmq</groupId>
        <artifactId>amqp-client</artifactId>
        <version>5.8.0</version>
    </dependency>
```

## 消息生产者

消息生产者：生产消息

```java
public class Producer {

    private final static String QUEUE_NAME = "Hello";

    public static void main(String[] args) throws Exception {
        // 1.创建连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("106.52.180.14");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("123");

        // 2.创建连接
        Connection connection = factory.newConnection();

        // 3.创建通道（实现了自动 close 接口 自动关闭 不需要显示关闭）
        Channel channel = connection.createChannel();

        /**
         * 4.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        /**
         * 5.发送消息
         * 参数1：交换机（不指定，使用默认交换机）
         * 参数2：路由键
         * 参数3：其他参数
         * 参数4：消息主体
         */
        String message = "hello world";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());

        System.out.println("消息发送完成~");
    }

}
```

## 消息消费者

消息消费者：消费消息

```java
public class Consumer {

    private final static String QUEUE_NAME = "Hello";

    public static void main(String[] args) throws Exception {
        // 1.创建连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("106.52.180.14");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("123");

        // 2.创建连接
        Connection connection = factory.newConnection();

        // 3.创建通道（实现了自动 close 接口 自动关闭 不需要显示关闭）
        Channel channel = connection.createChannel();

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("消费消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

## 执行结果

1）启动生产者，发送消息

启动生产者，发送消息：  
![生产者发送消息](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626949235794.jpg)

查看管理页面，可以发现：新建了一个队列：Hello，并且队列里有一条消息等待读取：  
![管理页面，查看队列](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626949728101.jpg)

进入队列详情，还可以看到消息主题内容：  
![管理页面，查看消息主体内容](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626949629441.jpg)

2）启动消费者，消费消息

启动消费者，消费消息：  
![消费者消费消息](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626949896540.jpg)

查看管理页面，可以发现：Hello队列，消息已经被消费了：  
![管理页面，查看队列](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626949947003.jpg)

## 小结

上述的入门案例中其实使用的是的简单模式。

![基本消息模式](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721223009.png)  
在上图的模式中，有以下概念：
1. 生产者（P）：也就是要发送消息的程序
2. 消费者（C）：消息的接受者，会一直等待消息到来。
3. 消息队列：图中红色部分。类似一个邮箱，可以缓存消息；生产者向其中投递消息，消费者从其中取出消息。

# 工作队列模式（work queue）

工作队列(又称任务队列)  
主要思想是避免立即执行资源密集型任务，必须等待它执行完成。  
相反我们稍后完成任务，我们将任务封装为消息并将其发送到队列，在后台运行的工作进程将获取任务并最终执行作业。当你运行许多消费者时，任务将在他们之间共享，但是一个消息只能被一个消费者获取。

## 图解

![工作消息模式](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721224055.png)  

- Work Queues：与基本消息模式相比，多了一个或多个消费者，多个消费者共同消费同一个队列中的消息，但是一个消息只能被一个消费者获取。
- 应用场景：对于任务过重或任务比较多情况使用工作队列可以提高任务处理的速度。

思考：当有多个消费者时，我们的消息会被哪个消费者消费呢，我们又该如何均衡消费者消费信息的多少呢？  
主要有两种模式：
1. 轮询分发：一个消费者一条，按均分配；
2. 不公平分发：根据消费者的消费能力进行分发，处理快的处理的多，处理慢的处理的少；

## 抽取工具类

```java
public class RabbitMqUtils {

    public static Channel getChannel() throws Exception {
        // 1.创建连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("106.52.180.14");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("123");

        // 2.创建连接
        Connection connection = factory.newConnection();

        // 3.创建通道（实现了自动 close 接口 自动关闭 不需要显示关闭）
        Channel channel = connection.createChannel();

        return channel;
    }

}
```

## 轮询分发

### 生产者

```java
public class Producer {

    private final static String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws Exception {
        // 获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 发送消息，模拟发送多个消息，测试工作队列
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println("发送消息：" + message);
        }
    }

}
```

### 消费者

```java
public class Consumer {

    private final static String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws Exception {
        // 获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, (consumerTag, message) -> {
            String msgBody = new String(message.getBody());
            System.out.println("消费消息，消息内容：" + msgBody);
        }, (consumerTag) -> {
            System.out.println("消费消息失败了~");
        });
    }

}
```

### 执行结果

1）启动两个消息者线程，模拟两个消费者在监听队列消息消息：  
![消费者1](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721232358.png)  
![消费者2](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721232422.png)  

2）启动生产者进行发送消息。

3）结论
通过生产者总共发送 4 个消息；  
消费者 1 和消费者 2 分别分得两个消息，并且是按照有序的一个接收一次消息：  
![工作消息模式](https://rong0624.gitee.io/images/MQ/RabbitMQ/20210721232715.png)  

## 不公平分发（能者多劳）

以上RabbitMQ 分发消息采用的轮询分发，但是在某种场景下这种策略并不是很好。  
比方说：有两个消费者在处理任务，其中有个消费者 1 处理任务的速度非常快，而另外一个消费者 2 处理速度却很慢，这个时候我们还是采用轮询分发的话，  
这处理速度快的这个消费者很大一部分时间处于空闲状态，而处理慢的那个消费者一直在干活，这种分配方式在这种情况下其实就不太好，但是 RabbitMQ 并不知道这种情况，它依然很公平的进行分发。

现在想要做的是：不公平分发，消费越快的人，消费的越多，怎么实现呢？在消费者指定prefetchCount。
```java
Integer prefetchCount = 1
channel.basicQos(prefetchCount);
```

![不公平分发图](https://rong0624.gitee.io/images/MQ/RabbitMQ/1626919721516.jpg)

解释：如果这个任务我还没有处理完或者我还没有应答你，你先别分配给我，我目前只能处理一个任务，  
然后 rabbitmq 就会把该任务分配给没有那么忙的那个空闲消费者，当然如果所有的消费者都没有完成手上任务，  
队列还在不停的添加新任务，队列有可能就会遇到队列被撑满的情况，这个时候就只能添加新的 worker 或者改变其他存储任务的策略。

## 小结

1. 在一个队列中，如果有多个消费者，那么消费者之间对同一个消息的关系是竞争关系，一个消息只能被一个消费者获取。
2. Work Queues 对于任务过重或任务比较多情况使用工作队列可以提高任务处理的速度。列如：关系服务部署多个，只需要一个节点成功发送即可。

# 订阅模式-Fanout

Fanout，也称为广播模式。在改模式下会将消息交给所有绑定到交换机的队列。

## 图解

![订阅模式-Fanout](https://rong0624.gitee.io/images/MQ/RabbitMQ/1630658875186.jpg)

- P：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给交换机（X）
- C：消费者，消息的接收者，会一直等待消息到来
- Queue：消息队列，接收消息、缓存消息
- Exchange：交换机（X），一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。
- Fanout类型交换机会将消息交给所有绑定到交换机的队列。

## 实现需求

同一个消息，通过订阅模式-Fanout被多个消费者消费。

## 消息生产者

变化：
```
1. 声明Exchange，不再声明Queue
2. 发送消息到指定Exchange，而不是默认交换机
```

```
public class Producer {

    private final static String EXCHANGE_NAME = "test.fanout";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明交换机
         * 参数1：交换机名称
         * 参数2：交换机类型
         * 参数3：是否持久化
         * 参数4：是否自动删除
         * 参数5：其他参数
         */
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, false, false, null);

        // 发送消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            /**
             * 3.发送消息
             * 参数1：交换机（不指定，使用默认交换机）
             * 参数2：路由键
             * 参数3：其他参数
             * 参数4：消息主体
             */
            channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
            System.out.println("发送消息：" + message);
        }
    }

}
```

## 消息消费者

变化：
```
1. 需要声明队列
2. 建立队列与交换机的绑定
```

消费者1：
```
public class Consumer1 {

    private final static String EXCHANGE_NAME = "test.fanout";
    private final static String QUEUE_NAME = "funout_consumer1";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为fanout，routingKey设置为""
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("控制台打印消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

消费者2：
```
public class Consumer2 {

    private final static String EXCHANGE_NAME = "test.fanout";
    private final static String QUEUE_NAME = "funout_consumer2";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为fanout，routingKey设置为""
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("消息保存到数据，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

## 执行结果

1）启动生产者

启动生产者后，可以看到我们自己声明的交换机  
![fanout-自己声明的交换机](https://rong0624.gitee.io/images/MQ/RabbitMQ/1630659575657.jpg)

2）启动消费者

启动两个消费者后，可以在交换机上看到队列与交换机的绑定关系  
![fanout-队列&交换机绑定关系](https://rong0624.gitee.io/images/MQ/RabbitMQ/1630659702681.jpg)

3）生产者发送消息，查看消费者消息情况

这边不演示了，当发送三条消息后，每个消费者都可以消费到三条消息。  
得出结论：每个队列里面的消息只能被消费一次，但可以通过订阅模式-funout下声明多个队列来多次消费同一条消息。

## 小结

在订阅-fanout模式下的小结：
1. 消费者可以有自己的Queue（多个队列）
2. 队列要绑定到Exchange（交换机）
3. 当交换机类型为fanout，会把消息发送给绑定过的所有队列（忽略路由键）
4. 在订阅-fanout模式下，多个队列可以存储同一个消息，实现一条消息被多个消费者消费
5. 订阅-fanout与工作队列模式的区别：
    - 发布/订阅模式需要定义交换机，而工作队列模式不用定义交换机（工作队列使用默认交换机）
    - 发布/订阅模式需要设置队列和交换机的绑定，工作队列不需要设置（工作队列实际上会自动将队列绑定到默认的交换机上）

# 订阅模式-Direct

Direct，也称为直连模式。在该模式下必须队列的绑定 Routing key 与消息的 Routing key 完全一致才能接受到消息。

订阅模式-Direct的约定：
- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个 RoutingKey（路由key）
- 消息的发送方在向 Exchange 发送消息时，也必须指定消息的 RoutingKey
- Exchange 不再把消息交给每一个绑定的队列，而是根据消息的 Routing Key 进行判断，只有队列的绑定 Routing key 与消息的 Routing key 完全一致，才会接收到消息

## 图解

![订阅模式-Direct图解](https://rong0624.gitee.io/images/MQ/RabbitMQ/1630664309047.jpg)  
- P：生产者，向 Exchange 发送消息，发送消息时，会指定一个routing key
- X：Exchange（交换机），接收生产者的消息，然后把消息递交给与 routing key 完全匹配的队列
- C1：消费者，其所在队列指定了需要 routing key 为 error 的消息
- C2：消费者，其所在队列指定了需要 routing key 为 info、error、warning 的消息

## 实现需求

生产者发送 info、error、warning 路由键消息。
消费者1消费所有的消息，消费者2只消费error的消息。

## 消息生产者

```java

public class Producer {

    private final static String EXCHANGE_NAME = "test.direct";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明交换机
         * 参数1：交换机名称
         * 参数2：交换机类型
         * 参数3：是否持久化
         * 参数4：是否自动删除
         * 参数5：其他参数
         */
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, false, false, null);

        // 发送消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            /**
             * 3.发送消息
             * 参数1：交换机（不指定，使用默认交换机）
             * 参数2：路由键
             * 参数3：其他参数
             * 参数4：消息主体
             */
            String routingKey;
            if (message.contains("info")) {
                routingKey = "info";
            } else if (message.contains("error")) {
                routingKey = "error";
            } else {
                routingKey = "warning";
            }
            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
            System.out.println("发送消息：" + message);
        }
    }

}
```

## 消息消费者

消费者1：接受 warning，info，error 路由键的消息
```java

public class Consumer1 {

    private final static String EXCHANGE_NAME = "test.direct";
    private final static String QUEUE_NAME = "direct_consumer1";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为direct，routingKey设置为想要接受的消息路由键
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "warning");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "info");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "error");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("控制台打印消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

消费者2：接受 error 路由键的消息
```java

public class Consumer2 {

    private final static String EXCHANGE_NAME = "test.direct";
    private final static String QUEUE_NAME = "direct_consumer2";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为direct，routingKey设置为想要接受的消息路由键
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "error");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("数据库保存消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

## 执行结果

1）启动消息生产者

启动生产者后，可以看到我们自己声明的交换机，并且交换机类型是：direct。
![声明direct类型交换机](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631175266353.jpg)

2）启动消息消费者

启动消费者1和消费者2，可以看到绑定关系。  
可以看到：消费者1等待接收warning，info，error路由键的消息，消费者2等待接收error路由键的消息。
![direct类型交换机绑定关系](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631175314170.jpg)

3）生产者发送消息，查看结果

生产者发送消息：  
![生产者发送消息](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631175556983.jpg)

消费者1消费情况：  
![消费者1消费情况](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631175568487.jpg)

消费者2消费情况：  
![消费者2消费情况](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631175573846.jpg)

生产者分别发送warning，info，error路由键消息内容，消费者1都接收到了，消费者2只接收到error路由键的消息。

## 小结

订阅模式-Direct要求队列在绑定交换机时要指定 routing key，消息会转发到符合 routing key 的队列。

# 订阅模式-Topic

订阅模式-Topic的约定：
- Topic 类型与 Direct 相比，都是可以根据 RoutingKey 把消息路由到不同的队列。只不过 Topic 类型Exchange 可以让队列在绑定 Routing key 的时候使用通配符！
- Routing key 中可以存在两种特殊字符 "\*”与“#”，用于做模糊匹配，其中 “\*” 用于匹配一个单词，“#”用于匹配一个或多个词（可以是零个）
- Routing key 一般都是有一个或多个单词组成，这些单词可以是任意单词，多个单词之间以”.”分割，例如"stock.usd.nyse", "nyse.vmw"

## 图解

![订阅模式-Topic图解](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631182623236.jpg)
- 红色 Queue：绑定的是 usa.# ，因此凡是以 usa. 开头的 routing key 都会被匹配到
- 黄色 Queue：绑定的是 #.news ，因此凡是以 .news 结尾的 routing key 都会被匹配

## 实现需求

生产者发送 info、error、warning 路由键消息 和 订单相关的日志。
消费者1消费所有（#.info.#，#.error.#，#.warning.#）的消息；
消费者2只消费error的消息和订单相关的日志（#.error.#，#.order.#）。

## 消息生产者

```java
public class Producer {

    private final static String EXCHANGE_NAME = "test.topic";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明交换机
         * 参数1：交换机名称
         * 参数2：交换机类型
         * 参数3：是否持久化
         * 参数4：是否自动删除
         * 参数5：其他参数
         */
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, false, false, null);

        // 发送消息
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()) {
            String message = scanner.next();
            /**
             * 3.发送消息
             * 参数1：交换机（不指定，使用默认交换机）
             * 参数2：路由键
             * 参数3：其他参数
             * 参数4：消息主体
             */
            String routingKey = "";
            if (message.contains("order")) {
                routingKey = "order.";
            }
            if (message.contains("info")) {
                routingKey += "info";
            } else if (message.contains("error")) {
                routingKey += "error";
            } else {
                routingKey += "warning";
            }
            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
            System.out.println("发送消息：" + message);
        }
    }

}
```

## 消息消费者

消费者1：接受 #.info.#，#.error.#，#.warning.# 路由键的消息
```java
public class Consumer1 {

    private final static String EXCHANGE_NAME = "test.topic";
    private final static String QUEUE_NAME = "topic_consumer1";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为direct，routingKey设置为想要接受的消息路由键
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "#.warning.#");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "#.info.#");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "#.error.#");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("控制台打印消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

消费者2：接受 #.error.#，#.order.# 路由键的消息
```java
public class Consumer2 {

    private final static String EXCHANGE_NAME = "test.topic";
    private final static String QUEUE_NAME = "topic_consumer2";

    public static void main(String[] args) throws Exception {
        // 1.获取通道
        Channel channel = RabbitMqUtils.getChannel();

        /**
         * 2.声明队列
         * 参数1：队列名称
         * 参数2：是否持久化队列，不持久化的队列重启访问后丢失
         * 参数3：是否独占队列
         * 参数4：是否自动删除队列，当最后一个消费者退订后即被删除
         * 参数5：其他参数
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        /**
         * 3.绑定（队列与交换机建立绑定）
         * 参数1：队列名称
         * 参数2：交换机名称
         * 参数3：路由键
         *          如果交换机类型为direct，routingKey设置为想要接受的消息路由键
         */
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "#.error.#");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "order.#");

        // 消费消息的程序
        DeliverCallback deliverCallback = (consumerTag, message) -> {
            // 模拟异常
            String msgBody = new String(message.getBody());
            System.out.println("数据库保存消息，消息内容：" + msgBody);
        };

        // 消费消息失败的程序
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消费消息失败了~");
        };

        /**
         * 4.接受消息
         * 参数1：监听的队列
         * 参数2：是否自动应答
         * 参数3：消费消息的程序
         * 参数4：消费消息失败的程序
         */
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, cancelCallback);
    }

}
```

## 执行结果

1）启动消息生产者

启动生产者后，可以看到我们自己声明的交换机，并且交换机类型是：topic。
![声明topic类型交换机](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631242891106.jpg)

2）启动消息消费者

启动消费者1和消费者2，可以看到绑定关系。  
![topic类型交换机绑定关系](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631242898127.jpg)

3）生产者发送消息，查看结果

生产者发送消息：  
![生产者发送消息](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631242903958.jpg)

消费者1消费情况：  
![消费者1消费情况](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631242910856.jpg)

消费者2消费情况：  
![消费者2消费情况](https://rong0624.gitee.io/images/MQ/RabbitMQ/1631242916251.jpg)

生产者分别发送 warning，info，error，订单相关 路由键消息内容（共6条）。
消费者1都接收到了（接受到共6条），消费者2只接收到 error 与 订单 相关路由键的消息（接受到共4条）。

## 小结

订阅-Topic模式可以实现 订阅-Funout模式 与 订阅-Direct模式 的功能，只是 订阅-Topic模式 在配置 routing key 的时候可以使用通配符，显得更加灵活。

# Exchange 交换机

## 概念

RabbitMQ 消息传递模式的核心思想是：生产者生产的消息从不会直接发送到队列。实际上，通常生产者甚至都不知道这些消息传递传递到了哪些队列中，生产者只能将消息发送到交换机(exchange)。

交换机工作的内容非常简单：一方面它接收来自生产者的消息，另一方面将它们推入队列。交换机必须确切知道如何处理收到的消息。是应该把这些消息放到特定队列还是说把他们到许多队列中还是说应该丢弃它们。这就的由交换机的类型来决定。

交换机是用来发送消息的AMQP实体。  
交换机拿到一个消息之后将它路由给一个或零个队列。  
交换机使用哪种路由算法是由交换机类型和绑定（Bindings）规则所决定的。  
交换机只负责转发消息，不具备存储消息的功能，如果没有符合路由规则的队列，那么消息会丢失。

## 交换机属性

在声明交换机时还可以附带许多其他的属性，其中最重要的几个分别是：  
- Name  
- Type（交换机类型）
- Durability（消息代理重启后，交换机是否还存在）  
- Auto-delete（当所有与之绑定的消息队列都完成了对此交换机的使用后，删除它）  
- Arguments（依赖代理本身）

## Binding 绑定

绑定（Binding）是交换机（exchange）将消息（message）路由给队列（queue）所需遵循的规则。
如果要指示交换机“E”将消息路由给队列“Q”，那么“Q”就需要与“E”进行绑定。绑定操作需要定义一个可选的路由键（routing key）属性给某些类型的交换机。路由键的意义在于从发送给交换机的众多消息中选择出某些消息，将其路由给绑定的队列。

打个比方：  
队列（queue）是我们想要去的位于纽约的目的地  
交换机（exchange）是JFK机场  
绑定（binding）就是JFK机场到目的地的路线。能够到达目的地的路线可以是一条或者多条  
拥有了交换机这个中间层，很多由发布者直接到队列难以实现的路由方案能够得以实现，并且避免了应用开发者的许多重复劳动。  

如果AMQP的消息无法路由到队列（例如，发送到的交换机没有绑定队列），消息会被就地销毁或者返还给发布者。如何处理取决于发布者设置的消息属性。

## 交换机类型

AMQP 0-9-1的代理提供了四种交换机：  
![交换机类型](https://rong0624.gitee.io/images/MQ/交换机类型.png)  

默认交换机，直连交换机，扇形交换机，主题交换机，头交换机；

## 默认交换机（default exchange）

### 概念

默认交换机（default exchange）实际上是一个由消息代理预先声明好的没有名字（名字为空字符串）的直连交换机（direct exchange）。

它有一个特殊属性使得它对于简单应用特别有用处：那就是每新建队列（queue）都会自动绑定到默认交换机上，绑定的路由键（routing key）名称与队列名称相同。

举个栗子：  
当你声明了一个名为"search-indexing-online"的队列，AMQP代理会自动将其绑定到默认交换机上，绑定（binding）的路由键名称也是为"search-indexing-online"。因此，当携带着名为"search-indexing-online"的路由键的消息被发送到默认交换机的时候，此消息会被默认交换机路由至名为"search-indexing-online"的队列中。换句话说，默认交换机看起来貌似能够直接将消息投递给队列，尽管技术上并没有做相关的操作。

### 实战

基本消息模式和工作队列消息模式案例中使用的都是默认交换机，路由键名是队列名称，所以消息才可以成功的被发送到队列中。
```java
/**
* 5.发送消息
* 参数1：交换机（不指定，使用默认交换机）
* 参数2：路由键
* 参数3：其他参数
* 参数4：消息主体
*/
String message = "hello world";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
```

## 扇型交换机（funout exchange）

### 概念

扇型交换机（funout exchange）将消息路由给绑定到它身上的所有队列，而不理会绑定的路由键。如果N个队列绑定到某个扇型交换机上，当有消息发送给此扇型交换机时，交换机会将消息的拷贝分别发送给这所有的N个队列。扇型用来交换机处理消息的广播路由（broadcast routing）。

扇型交换机图例：  
![扇型交换机图解](https://rong0624.gitee.io/images/MQ/扇型交换机图解.png)

上图所示，生产者（P）生产消息 1，并将消息 1 推送到 Exchange，由于 Exchange Type=fanout 这时候会遵循 fanout 的规则将消息推送到所有与它绑定 Queue，也就是图上的两个 Queue 最后两个消费者消费。

### 实战

请看：订阅模式-Fanout

## 直连交换机（direct exchange）

### 概念

直连交换机（direct exchange）是根据消息携带的路由键（routing key）将消息投递给对应队列的。直连交换机用来处理消息的单播路由（unicast routing）（尽管它也可以处理多播路由）。

下面介绍它是如何工作的：  
1）将一个队列绑定到某个交换机上，同时赋予该绑定一个路由键（routing key）  
2）当一个携带着路由键为R的消息被发送给直连交换机时，交换机会把它路由给绑定值同样为R的队列。

直连型交换机图例：  
![直连交换机图解](https://rong0624.gitee.io/images/MQ/直连交换机图解.png)

当生产者（P）发送消息时 Rotuing key=booking 时，这时候将消息传送给 Exchange，Exchange 获取到生产者发送过来消息后，会根据自身的规则进行与匹配相应的 Queue，这时发现 Queue1 和 Queue2 都符合，就会将消息传送给这两个队列。  
如果我们以 Rotuing key=create 和 Rotuing key=confirm 发送消息时，这时消息只会被推送到 Queue2 队列中，其他 Routing Key 的消息将会被丢弃。

### 实战

请看：订阅模式-Direct

## 主题交换机（topic exchanges）

### 概念

主题交换机（topic exchanges）通过对消息的路由键和队列到交换机的绑定模式之间的匹配，将消息路由给一个或多个队列。主题交换机经常用来实现各种分发/订阅模式及其变种。主题交换机通常用来实现消息的多播路由（multicast routing）。

主题交换机规则：  
前面提到的 direct 规则是严格意义上的匹配，换言之 Routing Key 必须与 Binding Key 相匹配的时候才将消息传送给 Queue.  
而Topic 的路由规则是一种模糊匹配，可以通过通配符满足一部分规则就可以传送。  

它的约定是：  
1）binding key 中可以存在两种特殊字符 “*” 与“#”，用于做模糊匹配，其中 “*” 用于匹配一个单词，“#”用于匹配多个单词（可以是零个）   
2）routing key 为一个句点号 “.” 分隔的字符串（我们将被句点号 “. ” 分隔开的每一段独立的字符串称为一个单词），如“stock.usd.> nyse”、“nyse.vmw”、“quick.> orange.rabbit”  
3）binding key 与 routing key 一样也是句点号 “.” 分隔的字符串

主题交换机图例：  
![主题交换机图解](https://rong0624.gitee.io/images/MQ/主题交换机图解.png)

当生产者发送消息 Routing Key=F.C.E 的时候，这时候只满足 Queue1，所以会被路由到 Queue 中，如果 Routing Key=A.C.E 这时候会被同是路由到 Queue1 和 Queue2 中，如果 Routing Key=A.F.B 时，这里只会发送一条消息到 Queue2 中。

### 实战

请看：订阅模式-Topic