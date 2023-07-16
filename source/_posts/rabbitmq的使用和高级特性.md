---
title: rabbitmq使用和高级特性
date: 2023-7-5
cover: https://s2.loli.net/2023/07/16/9wqUGTRdE6v1IZF.jpg
coverWidth: 1200
coverHeight: 750
tag:
   - rabbitmq
categories: 
   - rabbitmq

---








****1.单机部署****

****1.1.下载镜像****

方式一：在线拉取

```
docker pull rabbitmq:3-management
```

# *安装MQ*

执行下面的命令来运行MQ容器：

```
docker run \
 -e RABBITMQ_DEFAULT_USER=banyan \
 -e RABBITMQ_DEFAULT_PASS=lhj020826.. \
 --name mq \
 --hostname mq1 \
 -p 15672:15672 \
 -p 5672:5672 \
 -d \
 rabbitmq:3.12.1-management
```

**腾讯云开放端口**

需要开发两个端口，一个是管理界面，一个是发送消息界面

![Untitled](images/Untitled.png)

# **集群部署**

接下来，我们看看如何安装RabbitMQ的集群。

## **集群分类**

在RabbitMQ的官方文档中，讲述了两种集群的配置方式：

- 普通模式：普通模式集群不进行数据同步，每个MQ都有自己的队列、数据信息（其它元数据信息如交换机等会同步）。例如我们有2个MQ：mq1，和mq2，如果你的消息在mq1，而你连接到了mq2，那么mq2会去mq1拉取消息，然后返回给你。如果mq1宕机，消息就会丢失。
- 镜像模式：与普通模式不同，队列会在各个mq的镜像节点之间同步，因此你连接到任何一个镜像节点，均可获取到消息。而且如果一个节点宕机，并不会导致数据丢失。不过，这种方式增加了数据同步的带宽消耗。

我们先来看普通模式集群。

## **Docker集群**

### 获取cookie

RabbitMQ底层依赖于Erlang，而Erlang虚拟机就是一个面向分布式的语言，默认就支持集群模式。集群模式中的每个RabbitMQ 节点使用 cookie 来确定它们是否被允许相互通信。

要使两个节点能够通信，它们必须具有相同的共享秘密，称为Erlang cookie。cookie 只是一串最多 255 个字符的字母数字字符。

每个集群节点必须具有相同的 cookie。实例之间也需要它来相互通信。

我们先在之前启动的mq容器中获取一个cookie值，作为集群的cookie。执行下面的命令：

```docker
docker exec -it mq cat /var/lib/rabbitmq/.erlang.cookie
```

![Untitled](images/Untitled%201.png)

2.2.2创建相同的配置文件

![Untitled](images/Untitled%202.png)

内容如下

```docker
loopback_users.guest = false
listeners.tcp.default = 5672
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@mq1
cluster_formation.classic_config.nodes.2 = rabbit@mq2
cluster_formation.classic_config.nodes.3 = rabbit@mq3
```

### 创建一个相同的文件存放Cooke

![Untitled](images/Untitled%203.png)

### 创建n个文件夹来实现配置文件的存放例如

![Untitled](images/Untitled%204.png)

### 启动容器

```docker
docker run -d --net mq-net \
-v ${PWD}/mq1/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf \
-v ${PWD}/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie \
-e RABBITMQ_DEFAULT_USER=banyan \
-e RABBITMQ_DEFAULT_PASS=lhj020826.. \
--name mq1 \
--hostname mq1 \
-p 5672:5672 \
-p 15672:15672 \
rabbitmq:3.12.1-management
```

![Untitled](images/Untitled%205.png)

### 开启镜像模式

在刚刚的案例中，一旦创建队列的主机宕机，队列就会不可用。不具备高可用能力。如果要解决这个问题，必须使用官方提供的镜像集群方案。

官方文档地址：[https://www.rabbitmq.com/ha.html](https://www.rabbitmq.com/ha.html)

### 镜像模式的特征

默认情况下，队列只保存在创建该队列的节点上。而镜像模式下，创建队列的节点被称为该队列的主节点，队列还会拷贝到集群中的其它节点，也叫做该队列的镜像节点。

但是，不同队列可以在集群中的任意节点上创建，因此不同队列的主节点可以不同。甚至，一个队列的主节点可能是另一个队列的镜像节点。

用户发送给队列的一切请求，例如发送消息、消息回执默认都会在主节点完成，如果是从节点接收到请求，也会路由到主节点去完成。镜像节点仅仅起到备份数据作用。

当主节点接收到消费者的ACK时，所有镜像都会删除节点中的数据。

总结如下：

镜像队列结构是一主多从（从就是镜像）
所有操作都是主节点完成，然后同步给镜像节点
主宕机后，镜像节点会替代成新的主（如果在主从同步完成前，主就已经宕机，可能出现数据丢失）
不具备负载均衡功能，因为所有操作都会有主节点完成（但是不同队列，其主节点可以不同，可以利用这个提高吞吐量

### 镜像模式的配置

| ha-mode | ha-params | 效果 |
| --- | --- | --- |
| 准确模式exactly | 队列的副本量count | 集群中队列副本（主服务器和镜像服务器之和）的数量。count如果为1意味着单个副本：即队列主节点。count值为2表示2个副本：1个队列主和1个队列镜像。换句话说：count = 镜像数量 + 1。如果群集中的节点数少于count，则该队列将镜像到所有节点。如果有集群总数大于count+1，并且包含镜像的节点出现故障，则将在另一个节点上创建一个新的镜像。 |
| all | (none) | 队列在群集中的所有节点之间进行镜像。队列将镜像到任何新加入的节点。镜像到所有节点将对所有群集节点施加额外的压力，包括网络I / O，磁盘I / O和磁盘空间使用情况。推荐使用exactly，设置副本数为（N / 2 +1）。 |
| nodes | node names | 指定队列创建到哪些节点，如果指定的节点全部不存在，则会出现异常。如果指定的节点在集群中存在，但是暂时不可用，会创建节点到当前客户端连接到的节点。
这里我们以rabbitmqctl命令作为案例来讲解配置语法。 |

语法示例：

### exactly模式

```docker
rabbitmqctl set_policy ha-two "^two\." '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

rabbitmqctl set_policy：固定写法
ha-two：策略名称，自定义
"^two\."：匹配队列的正则表达式，符合命名规则的队列才生效，这里是任何以two.开头的队列名称
'{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}': 策略内容
"ha-mode":"exactly"：策略模式，此处是exactly模式，指定副本数量
"ha-params":2：策略参数，这里是2，就是副本数量为2，1主1镜像
"ha-sync-mode":"automatic"：同步策略，默认是manual，即新加入的镜像节点不会同步旧的消息。如果设置为automatic，则新加入的镜像节点会把主节点中所有消息都同步，会带来额外的网络开销

### all模式

```docker
rabbitmqctl set_policy ha-all "^all\." '{"ha-mode":"all"}'
```

ha-all：策略名称，自定义
"^all\."：匹配所有以all.开头的队列名
'{"ha-mode":"all"}'：策略内容
"ha-mode":"all"：策略模式，此处是all模式，即所有节点都会称为镜像节点

### nodes模式

```docker
rabbitmqctl set_policy ha-nodes "^nodes\." '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
```

rabbitmqctl set_policy：固定写法
ha-nodes：策略名称，自定义
"^nodes\."：匹配队列的正则表达式，符合命名规则的队列才生效，这里是任何以nodes.开头的队列名称
'{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}': 策略内容
"ha-mode":"nodes"：策略模式，此处是nodes模式
"ha-params":["rabbit@mq1", "rabbit@mq2"]：策略参数，这里指定副本所在节点名称

### 通过控制面板开启

在admin→**Policies下**

![Untitled](images/Untitled%206.png)

# **SpringAMQP**

SpringAMQP是基于RabbitMQ封装的一套模板，并且还利用SpringBoot对其实现了自动装配，使用起来非常方便。

SpringAmqp的官方地址：[https://spring.io/projects/spring-amqp](https://spring.io/projects/spring-amqp)

SpringAMQP提供了三个功能：

- 自动声明队列、交换机及其绑定关系
- 基于注解的监听器模式，异步接收消息
- 封装了RabbitTemplate工具，用于发送消息
- 

![Untitled](images/Untitled%207.png)

![Untitled](images/Untitled%208.png)

## **Basic Queue 简单队列模型**

在父工程mq-demo中引入依赖

```xml
<!--AMQP依赖，包含RabbitMQ-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### **消息发送**

首先配置MQ地址，在publisher服务的application.yml中添加配置：

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1 # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: test # 用户名
    password: test # 密码
```

然后在publisher服务中编写测试类SpringAmqpTest，并利用RabbitTemplate实现消息发送：

```java
package cn.itcast.mq.spring;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringAmqpTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSimpleQueue() {
        // 队列名称
        String queueName = "simple.queue";
        // 消息
        String message = "hello, spring amqp!";
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

### **消息接收**

首先配置MQ地址，在consumer服务的application.yml中添加配置：

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1 # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: test # 用户名
    password: test # 密码
```

然后在consumer服务的`cn.itcast.mq.listener`包中新建一个类SpringRabbitListener，代码如下：

```java
package cn.itcast.mq.listener;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class SpringRabbitListener {

    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueueMessage(String msg) throws InterruptedException {
        System.out.println("spring 消费者接收到消息：【" + msg + "】");
    }
}
```

### **测试**

启动consumer服务，然后在publisher服务中运行测试代码，发送MQ消息

## **WorkQueue**

Work queues，也被称为（Task queues），任务模型。简单来说就是**让多个消费者绑定到一个队列，共同消费队列中的消息**。

当消息处理比较耗时的时候，可能生产消息的速度会远远大于消息的消费速度。长此以往，消息就会堆积越来越多，无法及时处理。

此时就可以使用work 模型，多个消费者共同处理消息处理，速度就能大大提高了。

![Untitled](images/Untitled%209.png)

### **消息发送**

这次我们循环发送，模拟大量消息堆积现象。

在publisher服务中的SpringAmqpTest类中添加一个测试方法：

```java
/**
     * workQueue
     * 向队列中不停发送消息，模拟消息堆积。
     */
@Test
public void testWorkQueue() throws InterruptedException {
    // 队列名称
    String queueName = "simple.queue";
    // 消息
    String message = "hello, message_";
    for (int i = 0; i < 50; i++) {
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message + i);
        Thread.sleep(20);
    }
}
```

### **消息接收**

要模拟多个消费者绑定同一个队列，我们在consumer服务的SpringRabbitListener中添加2个新的方法：

```java
@RabbitListener(queues = "simple.queue")
public void listenWorkQueue1(String msg) throws InterruptedException {
    System.out.println("消费者1接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(20);
}

@RabbitListener(queues = "simple.queue")
public void listenWorkQueue2(String msg) throws InterruptedException {
    System.err.println("消费者2........接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(200);
}
```

注意到这个消费者sleep了1000秒，模拟任务耗时。

### **测试**

启动ConsumerApplication后，在执行publisher服务中刚刚编写的发送测试方法testWorkQueue。

可以看到消费者1很快完成了自己的25条消息。消费者2却在缓慢的处理自己的25条消息。

也就是说消息是平均分配给每个消费者，并没有考虑到消费者的处理能力。这样显然是有问题的。

### **能者多劳**

在spring中有一个简单的配置，可以解决这个问题。我们修改consumer服务的application.yml文件，添加配置：

```
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个消息
```

### **总结**

Work模型的使用：

- 多个消费者绑定到一个队列，同一条消息只会被一个消费者处理
- 通过设置prefetch来控制消费者预取的消息数量

## **发布/订阅**

发布订阅的模型如图：

![Untitled](images/Untitled%2010.png)

可以看到，在订阅模型中，多了一个exchange角色，而且过程略有变化：

- Publisher：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给X（交换机）
- Exchange：交换机，图中的X。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。Exchange有以下3种类型：
    - Fanout：广播，将消息交给所有绑定到交换机的队列
    - Direct：定向，把消息交给符合指定routing key 的队列
    - Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列
- Consumer：消费者，与以前一样，订阅队列，没有变化
- Queue：消息队列也与以前一样，接收消息、缓存消息。

**Exchange（交换机）只负责转发消息，不具备存储消息的能力**，因此如果没有任何队列与Exchange绑定，或者没有符合路由规则的队列，那么消息会丢失！

## **Fanout**

Fanout，英文翻译是扇出，我觉得在MQ中叫广播更合适。

![Untitled](images/Untitled%2011.png)

在广播模式下，消息发送流程是这样的：

- 1） 可以有多个队列
- 2） 每个队列都要绑定到Exchange（交换机）
- 3） 生产者发送的消息，只能发送到交换机，交换机来决定要发给哪个队列，生产者无法决定
- 4） 交换机把消息发送给绑定过的所有队列
- 5） 订阅队列的消费者都能拿到消息

我们的计划是这样的：

- 创建一个交换机 itcast.fanout，类型是Fanout
- 创建两个队列fanout.queue1和fanout.queue2，绑定到交换机itcast.fanout
- 

![Untitled](images/Untitled%2012.png)

### **声明队列和交换机**

Spring提供了一个接口Exchange，来表示所有不同类型的交换机：

![Untitled](images/Untitled%2013.png)

在consumer中创建一个类，声明队列和交换机：

```java
package cn.itcast.mq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutConfig {
    /**
     * 声明交换机
     * @return Fanout类型交换机
     */
    @Bean
    public FanoutExchange fanoutExchange(){
        return new FanoutExchange("itcast.fanout");
    }

    /**
     * 第1个队列
     */
    @Bean
    public Queue fanoutQueue1(){
        return new Queue("fanout.queue1");
    }

    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue1(Queue fanoutQueue1, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }

    /**
     * 第2个队列
     */
    @Bean
    public Queue fanoutQueue2(){
        return new Queue("fanout.queue2");
    }

    /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding bindingQueue2(Queue fanoutQueue2, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }
}
```

### **消息发送**

在publisher服务的SpringAmqpTest类中添加测试方法：

```java
@Test
public void testFanoutExchange() {
    // 队列名称
    String exchangeName = "itcast.fanout";
    // 消息
    String message = "hello, everyone!";
    rabbitTemplate.convertAndSend(exchangeName, "", message);
}
```

### **消息接收**

在consumer服务的SpringRabbitListener中添加两个方法，作为消费者：

```java
@RabbitListener(queues = "fanout.queue1")
public void listenFanoutQueue1(String msg) {
    System.out.println("消费者1接收到Fanout消息：【" + msg + "】");
}

@RabbitListener(queues = "fanout.queue2")
public void listenFanoutQueue2(String msg) {
    System.out.println("消费者2接收到Fanout消息：【" + msg + "】");
}
```

### **总结**

交换机的作用是什么？

- 接收publisher发送的消息
- 将消息按照规则路由到与之绑定的队列
- 不能缓存消息，路由失败，消息丢失
- FanoutExchange的会将消息路由到每个绑定的队列

声明队列、交换机、绑定关系的Bean是什么？

- Queue
- FanoutExchange
- Binding

## **Direct**

在Fanout模式中，一条消息，会被所有订阅的队列都消费。但是，在某些场景下，我们希望不同的消息被不同的队列消费。这时就要用到Direct类型的Exchange。

在Direct模型下：

![Untitled](images/Untitled%2014.png)

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个`RoutingKey`（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 `RoutingKey`。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的`Routing Key`进行判断，只有队列的`Routingkey`与消息的 `Routing key`完全一致，才会接收到消息

**案例需求如下**：

1. 利用@RabbitListener声明Exchange、Queue、RoutingKey
2. 在consumer服务中，编写两个消费者方法，分别监听direct.queue1和direct.queue2
3. 在publisher中编写测试方法，向itcast. direct发送消息

![Untitled](images/Untitled%2015.png)

### **基于注解声明队列和交换机**

基于@Bean的方式声明队列和交换机比较麻烦，Spring还提供了基于注解方式来声明。

在consumer的SpringRabbitListener中添加两个消费者，同时基于注解来声明队列和交换机：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue1"),
    exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "blue"}
))
public void listenDirectQueue1(String msg){
    System.out.println("消费者接收到direct.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue2"),
    exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "yellow"}
))
public void listenDirectQueue2(String msg){
    System.out.println("消费者接收到direct.queue2的消息：【" + msg + "】");
}
```

### **消息发送**

在publisher服务的SpringAmqpTest类中添加测试方法：

```java
@Test
public void testSendDirectExchange() {
    // 交换机名称
    String exchangeName = "itcast.direct";
    // 消息
    String message = "红色警报！日本乱排核废水，导致海洋生物变异，惊现哥斯拉！";
    // 发送消息
    rabbitTemplate.convertAndSend(exchangeName, "red", message);
}
```

### **总结**

描述下Direct交换机与Fanout交换机的差异？

- Fanout交换机将消息路由给每一个与之绑定的队列
- Direct交换机根据RoutingKey判断路由给哪个队列
- 如果多个队列具有相同的RoutingKey，则与Fanout功能类似

基于@RabbitListener注解声明队列和交换机有哪些常见注解？

- @Queue
- @Exchange

## **Topic**

### **说明**

`Topic`类型的`Exchange`与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。只不过`Topic`类型`Exchange`可以让队列在绑定`Routing key` 的时候使用通配符！

`Routingkey` 一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： `item.insert`

通配符规则：

`#`：匹配一个或多个词

- ``：匹配不多不少恰好1个词

举例：

`item.#`：能够匹配`item.spu.insert` 或者 `item.spu`

`item.*`：只能匹配`item.spu`

图示：

![Untitled](images/Untitled%2016.png)

解释：

- Queue1：绑定的是`china.#` ，因此凡是以 `china.`开头的`routing key` 都会被匹配到。包括china.news和china.weather
- Queue2：绑定的是`#.news` ，因此凡是以 `.news`结尾的 `routing key` 都会被匹配。包括china.news和japan.news

案例需求：

实现思路如下：

1. 并利用@RabbitListener声明Exchange、Queue、RoutingKey
2. 在consumer服务中，编写两个消费者方法，分别监听topic.queue1和topic.queue2
3. 在publisher中编写测试方法，向itcast. topic发送消息

![Untitled](images/Untitled%2017.png)

### **消息发送**

在publisher服务的SpringAmqpTest类中添加测试方法：

```java
/**
     * topicExchange
     */
@Test
public void testSendTopicExchange() {
    // 交换机名称
    String exchangeName = "itcast.topic";
    // 消息
    String message = "喜报！孙悟空大战哥斯拉，胜!";
    // 发送消息
    rabbitTemplate.convertAndSend(exchangeName, "china.news", message);
}
```

### **消息接收**

在consumer服务的SpringRabbitListener中添加方法：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue1"),
    exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
    key = "china.#"
))
public void listenTopicQueue1(String msg){
    System.out.println("消费者接收到topic.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue2"),
    exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
    key = "#.news"
))
public void listenTopicQueue2(String msg){
    System.out.println("消费者接收到topic.queue2的消息：【" + msg + "】");
}
```

### **总结**

描述下Direct交换机与Topic交换机的差异？

- Topic交换机接收的消息RoutingKey必须是多个单词，以 `*.**` 分割
- Topic交换机与队列绑定时的bindingKey可以指定通配符
- `#`：代表0个或多个词
- ``：代表1个词

## **消息转换器**

之前说过，Spring会把你发送的消息序列化为字节发送给MQ，接收消息的时候，还会把字节反序列化为Java对象。

![Untitled](images/Untitled%2018.png)

只不过，默认情况下Spring采用的序列化方式是JDK序列化。众所周知，JDK序列化存在下列问题：

- 数据体积过大
- 有安全漏洞
- 可读性差

我们来测试一下。

### **测试默认转换器**

我们修改消息发送的代码，发送一个Map对象：

```java
@Test
public void testSendMap() throws InterruptedException {
    // 准备消息
    Map<String,Object> msg = new HashMap<>();
    msg.put("name", "Jack");
    msg.put("age", 21);
    // 发送消息
    rabbitTemplate.convertAndSend("simple.queue","", msg);
}
```

停止consumer服务

发送消息后查看控制台：

![Untitled](images/Untitled%2019.png)

### **配置JSON转换器**

显然，JDK序列化方式并不合适。我们希望消息体的体积更小、可读性更高，因此可以使用JSON方式来做序列化和反序列化。

在publisher和consumer两个服务中都引入依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
    <version>2.9.10</version>
</dependency>
```

配置消息转换器。

在启动类中添加一个Bean即可：

```java
@Bean
public MessageConverter jsonMessageConverter(){
    return new Jackson2JsonMessageConverter();
}
```

## 延时队列

分为插件延时队列和死信队列这种形式

### 插件延迟队列

### 插件的安装

访问 [Rabbitmq的github网址](https://github.com/orgs/rabbitmq/repositories?q&#61;delay&amp;type&#61;all&amp;language&#61;&amp;sort&#61;),检索 **delay** 找到插件`rabbitmq-delayed-message-exchange` ，如下图所示：

![Untitled](images/Untitled%2020.png)

下载mq对应的版本

下载ez文件

![Untitled](images/Untitled%2021.png)

上传到Linux

![Untitled](images/Untitled%2022.png)

通过docker命令将文件上传到docker的plugins目录下

```java

docker cp rabbitmq_delayed_message_exchange-3.12.0.ez f333139cfa22:/plugins
```

再进入到docker容器中

```java
docker exec -it 容易名称 /bin/bash
```

开启插件命令

```java
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

### 创建延迟交换机和延迟队列

```java
@Bean
    public Queue delayQueue1(){
        return new Queue("delay.queue1",false,false,false,null);
    }

    @Bean
    public Queue delayQueue2(){
        return new Queue("delay.queue2",false,false,false,null);
    }

    @Bean
    public CustomExchange delayExchange(){
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange("delay.exchange", "x-delayed-message", true, false, args);
    }

    @Bean
    public Binding bindingDelayExchange1(Queue delayQueue1, CustomExchange delayExchange) {
        return BindingBuilder.bind(delayQueue1).to(delayExchange).with("delay.routingKey").noargs();
    }
    @Bean
    public Binding bindingDelayExchange2(Queue delayQueue2, CustomExchange delayExchange) {
        return BindingBuilder.bind(delayQueue2).to(delayExchange).with("delay.routingKey2").noargs();
    }
```

### 生产者

```java
public void testDelayQueue() {
        // 交换机名称
        String exchangeName = "delay.exchange";
        // 消息
        String message = "hello, delay!";

        String routingKey = "delay.routingKey";
        // 发送消息
        rabbitTemplate.convertAndSend(exchangeName, routingKey, message, msg -> {
            msg.getMessageProperties().setDelay(3000);·
            return msg;
        });
        log.info("当前时间：{}，发送一条延迟{}毫秒的信息给队列delay.queue:{}", new Date(), 3000, message);
    }
```

### 消费者

```java
@RabbitListener(queues = "delay.queue1")
    public void listenDelayQueue1(String msg){
        log.info("当前时间：{}，收到一条延迟毫秒的信息给队列delay1.queue:{}", new Date(),msg);
    }
    @RabbitListener(queues = "delay.queue2")
    public void listenDelayQueue2(String msg){
        log.info("当前时间：{}，收到一条延迟毫秒的信息给队列delay2.queue:{}", new Date(),msg);
    }
```

## 死信队列

普通队列也有可能变成死信

注意，普通队列也需要绑定死信队列才能收到死信，不然会被丢弃

通过加入参数的形式

```java
public Queue topicQueue3(){
        HashMap<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", "dead.exchange");
        args.put("x-dead-letter-routing-key", "dead.routingKey");
        return new Queue("topic.queue3",false,false,false,args);
    }
```

死信就是消息在特定场景下的一种表现形式，这些场景包括：

消息被拒绝(basic.reject / basic.nack)，并且requeue = false

```java
System.out.println("消费者接收到topic.queue3的消息：【" + msg + "】");
        int idx = 0;
        while (idx < MAX_RETRIES) {
            idx++;
            String errorTip = "第" + idx + "次消费失败" +
                    ((idx < 3) ? "," + RETRY_INTERVAL + "s后重试" : ",进入死信队列");
            log.error(errorTip);
            Thread.sleep(RETRY_INTERVAL * 1000);
        }
       channel.basicNack(msg.getMessageProperties().getDeliveryTag(), false, false);
        log.info("消息已经拒绝,进入死信队列");
```

控制台输出

![Untitled](images/Untitled%2023.png)

—具体案例可以通过
消息的 TTL 过期时
消息队列达到最大长度
达到最大重试限制

在收到消息代码中直接抛出异常  `throw new RuntimeException("消息消费失败");`

可以看到先消费三次

![Untitled](images/Untitled%2024.png)

抛出异常后可以看到死信队列拿到了消息

![Untitled](images/Untitled%2025.png)

消息在这些场景中时，被称为死信。

![Untitled](images/Untitled%2026.png)

基本流程

需要的是死信交换机死信队列和延迟队列和延迟交换机

值得注意的是这个延迟交换机只是普通的路由交换机

```java
@Bean
    public DirectExchange delayExchange2(){
        return new DirectExchange("delay.exchange2");
    }

    @Bean
    public Queue delayQueue3(){
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", "dead.exchange");
        args.put("x-dead-letter-routing-key", "dead.routingKey");
        return new Queue("delay.queue3",false,false,false,args);
    }

    @Bean
    public Binding bindingDelayExchange3(Queue delayQueue3, DirectExchange delayExchange2) {
        return BindingBuilder.bind(delayQueue3).to(delayExchange2).with("delay.routingKey3");
    }
    /*
    死信交换机
     */
    @Bean
    public DirectExchange deadExchange(){
        return new DirectExchange("dead.exchange");
    }
    @Bean
    public Queue deadQueue(){
        return new Queue("dead.queue");
    }

    @Bean
    public Binding bindingDeadExchange(Queue deadQueue, DirectExchange deadExchange) {
        return BindingBuilder.bind(deadQueue).to(deadExchange).with("dead.routingKey");
    }
```

### 生产者

```java
public void testDeadQueue() throws InterruptedException {
        String exchangeName = "delay.exchange2";
        String routingKey = "delay.routingKey3";
        String message = "hello, dead!";
        String ttl = "5000";
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("消息发送成功:correlationData({}),ack({}),cause({})", correlationData, ack, cause);
            } else {
                log.info("消息发送失败:correlationData({}),ack({}),cause({})", correlationData, ack, cause);
            }
        });
        rabbitTemplate.setReturnCallback((message1, replyCode, replyText, exchange, routingKey1) -> {
            log.info("消息丢失:exchange({}),route({}),replyCode({}),replyText({}),message:{}", exchange, routingKey1, replyCode, replyText, message1);
        });
        rabbitTemplate.convertAndSend(exchangeName, routingKey, message, msg -> {
            msg.getMessageProperties().setExpiration(ttl);
            return msg;
        });
        log.info("当前时间：{}，发送一条延迟{}毫秒的信息给队列delay.queue:{}", new Date(), 5000, message);
        Thread.sleep(1000);
    }
```

### 消费者

```java
@RabbitListener(queues = "dead.queue")
    public void listenDeadQueue(Message msg){
        log.info("当前时间：{}，收到一条延迟毫秒的信息给队列delay2.queue:{}", new Date(),msg);
    }
```

值得注意的是，这两种方式时间的设置不太一样，方法有区别 `setDelay` 和 `setExpiration` 并且发送是向延迟队列发，消费是监听死信队列

#  RabbitMQ 高级特性

### 消息的可靠投递

在使用 RabbitMQ 的时候，作为消息发送方希望杜绝任何消息丢失或者投递失败场景。RabbitMQ 为我们提
供了两种方式用来控制消息的投递可靠性模式。
⚫ confirm 确认模式
⚫ return 退回模式
rabbitmq 整个消息投递的路径为：
producer--->rabbitmq broker--->exchange--->queue--->consumer
⚫ 消息从 producer 到 exchange 则会返回一个 confirmCallback 。
⚫ 消息从 exchange-->queue 投递失败则会返回一个 returnCallback 。
我们将利用这两个 callback 控制消息的可靠性投递

### 首先在配置文件中开启配置

```yaml
spring:
  rabbitmq:
    publisher-returns: true
    publisher-confirm-type: correlated
```

### ****ConfirmCallback确认模式****

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("消息发送成功:correlationData({}),ack({}),cause({})", correlationData, ack, cause);
            } else {
                log.info("消息发送失败:correlationData({}),ack({}),cause({})", correlationData, ack, cause);
            }
        });
rabbitTemplate.convertAndSend(exchangeName, routingKey, message, msg -> {
            msg.getMessageProperties().setExpiration(ttl);
            return msg;
        });
```

在生产者发送消息进行回调

说明：ConfirmCallback模式确认，需要重写confirm接方法，此方法的三个参数分别为：CorrelationData 、ack、cause

`CorrelationData`:对象内部只有一个id属性，用来表示当前消息的唯一性。
`ack` :消息投递状态，true表示投递成功
`cause`: 消息投递失败原因
虽然消息被broker接收到只能表示已经到达MQ服务器，但是并不能保证消息一定会被投递到目标 queue里。所以我们需要实现returnCallback来进行相关处理。

### ****ReturnCallback退回模式****

说明：实现接口ReturnCallback重写returnedMessage()方法，方法有五个参数message（消息体）、replyCode（响应code）、replyText（响应内容）、exchange（交换机）、routingKey（路由键）。

### 消息确认的三种模式

分别为basicAck、basicNack、basicReject。

basicAck模式

表示成功确认，使用此回执方法后，消息会被rabbitmq broker删除。

```java
void basicAck(long deliveryTag, boolean multiple)
```

- deliveryTag：消息投递序号，
- multiple：是否批量确认，值为 true则会一次性ack所有小于当前消息deliveryTag的消息。

basicNack模式

表示失败确认，一般在消费消息异常时用到此方法，可以将消息重新投递入队列。

```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
```

- deliveryTag：表示消息投递序号。
- requeue： 表示消息是否重新入队列，true表示重新投入队列中。
- multiple：是否批量确认，true表示会一次性ack所有小于当前消息deliveryTag的消息。

basicReject模式

basicReject：拒绝消息，与basicNack区别在于不能进行批量操作，其他用法很相似。

```java
void basicReject(long deliveryTag, boolean requeue)
```

- deliveryTag：消息投递序号。
- requeue：值为true表示消息重新入队列

### 消息可靠性保障

通过消息补偿机制来实现

![Untitled](images/Untitled%2027.png)

### 消息幂等性保障

全局唯一ID + (Redis/DB)
生产者在发送消息时，为每条消息设置一个全局唯一的messageId，消费者拿到消息后:
a、使用setnx命令，将messageId作为key放到redis中：setnx(messageId,1)，若返回1，说明之前没有消费过，正常消费；若返回0，说明这条消息之前已消费过，抛弃。
b、将全局消息ID存入数据库，若添加成功则说明首次执行，否则为重复消费

### 通过设置消息的messageid

```java
rabbitTemplate.convertAndSend(exchangeName, routingKey, message, msg -> {
            msg.getMessageProperties().setExpiration(ttl);
            msg.getMessageProperties().setMessageId("123456");
            return msg;
        });
```

### 消费端使用redisson来实现消息的消费