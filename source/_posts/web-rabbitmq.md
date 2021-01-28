---
title: SpringBoot中使用RabbitMQ详解
date: 2021-01-27 23:01:12
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/elk-0931238.png
categories:
  - spring boot
tags:
  - rabbitmq
  - emqp
---

# SpringBoot 中使用 RabbitMQ 详解

## 1. 简介

消息队列在后端应用中的作用主要有三点：

1. 解耦：可以用消息队列来解耦及时性要求不高的业务，让系统更灵活
2. 异步：异步处理重要性不高的业务逻辑，加快响应速度，比如注册时发邮件、发短信等
3. 削峰：对于有突发性大流量的业务，可以用消息中间件来削峰，挤压的请求放在消息队列，消费者慢慢处理

常见的消息中间件有：

ActiveMQ、RabbitMQ、RocketMQ、Kafka

- 其中 ActiveMQ 比较老，已经基本没人用了，不推荐使用

- RabbitMQ 社区活跃，使用人数很多，缺点是使用 erlang 语言开发的，不利于Java程序员定制开发，推荐小企业使用

- RocketMQ 是阿里开源的，基于Java语言，但是社区活跃度一般，可能哪天就不维护了，适用于技术实力强的企业

- Kafka 适合涉及实时计算和日志采集等场景使用，天然支持分布式，是业界标准

今天我们主要讲解的是 RabbitMQ

## 2. 基本概念

RabbitMQ 实现了 AMQP（Advanced Message Queuing Protocol）协议，下面来简单介绍下 AMQP 模型中的几个概念

|     名词     |   翻译   |                             解释                             |
| :----------: | :------: | :----------------------------------------------------------: |
|   Message    |   消息   | 由消息头和消息体组成，消息头包括 routing-key、priority、delivery-mode |
|  Publisher   |  生产者  |                          消息的来源                          |
|   Exchange   |  交换器  | 将生产者消息路由给服务器中的队列，类型有direct(默认)，fanout, topic, 和headers |
|    Queue     | 消息队列 |                   保存消息直到发送给消费者                   |
|   Binding    |   绑定   |                用于消息队列和交换器之间的关联                |
|  Connection  |   连接   |                       比如一个TCP连接                        |
|   Consumer   |  消费者  |                         消息的接收者                         |
| Virtual Host | 虚拟主机 | 表示一批交换器、消息队列和相关对象；vhost必须连接时指定；RabbitMQ的vhost默认是/ |
|    Broker    |   代理   |                      消息队列服务器实体                      |

消息发送简单流程：

![Publish path from publisher to consumer via exchange and queue](https://riger.oss-cn-shanghai.aliyuncs.com/img/hello-world-example-routing.png)

如图，`Publisher` 产出的消息发送到 `Exchange` , `Exchange`根据路由规则将消息发送到对应的队列 `Queue`，`Consumer`最后从队列中获取到消息

Exchange类型：

1. direct

点对点模式，消息中的路由键（routing key）如果和 Binding 中的 bindingkey 一致， 交换器就将消息发到对应的队列中。

2. fanout

广播模式，每个发到 fanout 类型交换器的消息都会分到所有绑定的队列上去

3. topic

将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。它将路由键和绑定键的字符串切分成单词，这些单词之间用点隔开。
识别通配符： # 匹配 0 个或多个单词， *匹配一个单词

4. Headers

   匹配消息内容中的 headers 属性，性能很差，不适用，基本看不到它的使用

routingkey 与 bindingkey 这两个概念比较容易混淆，这里详细解析下：

Exchange 和 Queue 都可以独立创建，创建完后是没有联系的。而上面提过消息是由 Exchange 分发到 Queue 的，所以我们要把 Exchange 与 Queue绑定，绑定的时候则必须**指定 bindingkey**。比如我们 绑定一个 Queue 到一个 topic Exchange，指定路由键为 `com.#`。我们发消息到 MQ 的时候必须**指定 routingkey**，比如我们指定一个 `com.rigerwu`就能匹配之前的 bindingkey，这条消息就能发送到这个 Queue。

## 3. 安装与配置

我写这篇文章的时候 RabbitMQ最新版本为：3.8.11

```bash
# 我们使用带管理界面的镜像
docker pull rabbitmq:3-management
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

默认的用户密码是：guest/guest

如果想改密码：（当然，因为我们使用的是`management`镜像，可以直接从界面的 `Admin` 标签里操作，这里记录一下命令行操作方式）

```bash
# 进入容器
docker exec -it rabbitmq /bin/bash
# 查看用户
rabbitmqctl list_users
# 修改密码
rabbitmqctl change_password userName newPassword
# 新增用户
rabbitmqctl add_user userName newPassword
# 删除用户
rabbitmqctl delete_user guest
# 设置自己账号为超级管理员
rabbitmqctl set_user_tags userName administrator
```

依赖引入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

> 使用 AmqpAdmin 可以声明 Exchange、Queue 以及它们的 Binding 
>
> 使用 RabbitTemplate 可以很方便得收发消息

配置文件，这里我们使用一个 SpringBoot 工程来演示即可:

```yaml
spring:
  application:
    name: web-rabbitmq
  rabbitmq:
    addresses: localhost
    port: 5672 # 默认值可不填
    username: guest # 默认值可不填
    password: guest # 默认值可不填
    virtual-host: / # 默认值可不填
```

## 4. 几种Exchange测试

我们创建3种 Exchange，3个 Queue，diret 直接用队列名作为 bindingkey，topic 分别用三种bindingkey绑定，如图所示：

![web-rabbitmq](https://riger.oss-cn-shanghai.aliyuncs.com/img/web-rabbitmq-1717482.png)

我们定义一个常量类：

```java
package com.rigerwu.web.rabbitmq.constants;

/**
 * created by riger on 2021/1/25
 */
public interface MQConsts {

    String DIRECT_EXCHANGE = "direct-exchange";
    String FANOUT_EXCHANGE = "fanout-exchange";
    String TOPIC_EXCHANGE = "topic-exchange";

    String QUEUE1 = "queue1";
    String QUEUE2 = "queue2";
    String QUEUE3 = "queue3";

    /**
     * * 一个占位符 #一个或者多个占位符
     */
    String QUEUE_ROUTING_KEY1 = "*.queue";
    String QUEUE_ROUTING_KEY2 = "#.queue";
    String QUEUE_ROUTING_KEY3 = "com.#";

}
```

定义一个配置文件，这是用来声明和绑定队列，也可以在 rabbitmq管理界面里做，代码做的好处是，如果没有会自动创建

```java
package com.rigerwu.web.rabbitmq.config;

import com.rigerwu.web.rabbitmq.constants.MQConsts;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.*;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

/**
 * created by riger on 2021/1/25
 */
@Configuration
@Slf4j
public class MQConfig {

    @Autowired
    private AmqpAdmin amqpAdmin;

    /**
     * 消息使用Json转换
     */
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @PostConstruct
    public void init() {
        DirectExchange direct = new DirectExchange(MQConsts.DIRECT_EXCHANGE);
        FanoutExchange fanout = new FanoutExchange(MQConsts.FANOUT_EXCHANGE);
        TopicExchange topic = new TopicExchange(MQConsts.TOPIC_EXCHANGE);
        amqpAdmin.declareExchange(direct);
        amqpAdmin.declareExchange(fanout);
        amqpAdmin.declareExchange(topic);
        Queue queue1 = new Queue(MQConsts.QUEUE1);
        Queue queue2 = new Queue(MQConsts.QUEUE2);
        Queue queue3 = new Queue(MQConsts.QUEUE3);
        amqpAdmin.declareQueue(queue1);
        amqpAdmin.declareQueue(queue2);
        amqpAdmin.declareQueue(queue3);
        // direct 这里偷懒直接用队列名作为 BindingKey
        amqpAdmin.declareBinding(BindingBuilder.bind(queue1).to(direct).with(MQConsts.QUEUE1));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue2).to(direct).with(MQConsts.QUEUE2));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue3).to(direct).with(MQConsts.QUEUE3));
        // fanout 可以看到 fanout exchange 绑定不需要 bindingkey
        amqpAdmin.declareBinding(BindingBuilder.bind(queue1).to(fanout));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue2).to(fanout));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue3).to(fanout));
        // topic
        amqpAdmin.declareBinding(BindingBuilder.bind(queue1).to(topic).with(MQConsts.QUEUE_ROUTING_KEY1));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue2).to(topic).with(MQConsts.QUEUE_ROUTING_KEY2));
        amqpAdmin.declareBinding(BindingBuilder.bind(queue3).to(topic).with(MQConsts.QUEUE_ROUTING_KEY3));
    }
}
```

定义一个简单的消息实体类：

```java
package com.rigerwu.web.rabbitmq.entity;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

/**
 * created by riger on 2021/1/25
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class MsgBean implements Serializable {

    private static final long serialVersionUID = -1248468460780960866L;
    private String msgDesc;
}
```

为每个队列关联一个接收者，打印消息：

```java
package com.rigerwu.web.rabbitmq.service;

import cn.hutool.json.JSONUtil;
import com.rigerwu.web.rabbitmq.constants.MQConsts;
import com.rigerwu.web.rabbitmq.entity.MsgBean;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;

/**
 * created by riger on 2021/1/25
 */
@Service
@Slf4j
public class QueueListener {

    @RabbitListener(queues = MQConsts.QUEUE1)
    @RabbitHandler
    public void receiveQueue1(MsgBean msgBean) {
        log.info("queue1 receive msg -> :" + JSONUtil.toJsonStr(msgBean));
    }

    @RabbitListener(queues = MQConsts.QUEUE2)
    @RabbitHandler
    public void receiveQueue2(MsgBean msgBean) {
        log.info("queue2 receive msg -> :" + JSONUtil.toJsonStr(msgBean));
    }

    @RabbitListener(queues = MQConsts.QUEUE3)
    @RabbitHandler
    public void receiveQueue3(MsgBean msgBean) {
        log.info("queue3 receive msg -> :" + JSONUtil.toJsonStr(msgBean));
    }
}
```

最后我们写一个 Controller 来测试：

```java
package com.rigerwu.web.rabbitmq.controller;

import cn.hutool.core.thread.ThreadUtil;
import com.rigerwu.web.rabbitmq.constants.MQConsts;
import com.rigerwu.web.rabbitmq.entity.MsgBean;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * created by riger on 2021/1/12
 */
@RestController
@Slf4j
public class MessageController {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @GetMapping("/direct")
    public String directMessage() {
        rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "queue1", new MsgBean("direct message 1"));
        rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "queue2", new MsgBean("direct message 2"));
        rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "queue3", new MsgBean("direct message 3"));
        rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "queue4", new MsgBean("direct message 4"));
        return "OK";
    }

    @GetMapping("/fanout")
    public String fanoutMessage() {
        rabbitTemplate.convertAndSend(MQConsts.FANOUT_EXCHANGE, "xxx", new MsgBean("fanout message"));
        return "OK";
    }

    @GetMapping("/topic")
    public String topicMessage() {

        log.info("queue1 bindingkey: *.queue");
        log.info("queue2 bindingkey: #.queue");
        log.info("queue3 bindingkey: com.#");

        String routingKey1 = "test.queue";
        String routingKey2 = "com.riger.queue";
        String routingKey3 = "com.queue";
        rabbitTemplate.convertAndSend(MQConsts.TOPIC_EXCHANGE, routingKey1,
                new MsgBean("topic message routingkey: "+ routingKey1));
        ThreadUtil.sleep(1000);
        log.info("----------------------------------------------");
        rabbitTemplate.convertAndSend(MQConsts.TOPIC_EXCHANGE, routingKey2,
                new MsgBean("topic message routingkey: "+ routingKey2));
        ThreadUtil.sleep(1000);
        log.info("----------------------------------------------");
        rabbitTemplate.convertAndSend(MQConsts.TOPIC_EXCHANGE, routingKey3,
                new MsgBean("topic message routingkey: "+ routingKey3));
        return "OK";
    }
}
```

direct 交换器，分别用 routingkey `queue1 queue2 queue3 queue4` 发送4条消息：

![image-20210127112529317](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127112529317.png)

可以看到前3条 routingkey 与 bindingkey 匹配，分别发送到3个队列，第4条则没有匹配，丢失

fanout 交换器，我们随意指定 `xxx`的 routingkey 发送一条消息，3个队列都接收到了，符合预期：

![image-20210127113033575](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127113033575.png)

topic 我们之前给3个队列分别绑定了 `*.queue  .queue  com.#`三种 bindingkey，发送的时候我们发送的是 `test.queue  com.riger.queue  com.queue`3种routingkey

> 这里两种key都要用点 `.`分隔单词
>
> `*`匹配一个单词
>
> `#`匹配0或者多个单词

![image-20210127113110375](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127113110375.png)

可以看到，`test.queue`能匹配队列1和2，`com.riger.queue`能匹配2和3，`com.queue`3个队列都能匹配，符合预期

## 5. 消息的可靠投递

消息的可靠投递其实说白了就是保证消息不丢，那么就要从三个方面来说：

1. 生产者不丢
2. MQ不丢
3. 消费者不丢

### 1. 生产者可靠性

生产者有可能弄丢消息，比如发消息给MQ，发出去，网络有问题，MQ没收到，如果生产者不管了，这条消息就丢了

要保证生产者发送消息的可靠性，一般有两种方法：

1. 开启事务
2. 开启confirm模式

开启事务会严重降低MQ的吞吐量，生产上不会去使用，一般我们都使用的 confirm 模式

confirm 开启方法，在生产者服务的 yml 里添加：

```yaml
spring:
  rabbitmq:
    # publisher-confirms: true # 这个设置已经过时,使用publisher-confirm-type
    # 开启消息确认可选有 none corelated simple, 消息发送到broker后回调
    publisher-confirm-type: correlated
    # 消息投递队列失败后回调 returnscallback 要配合mandatory参数使用
    publisher-returns: true
    template:
      # false 消息如果发送不到合适的队列会被丢弃
      # true 消息如果发送不到合适的队列会回调 returnscallback
      mandatory: true
```

`publisher-confirm-type` 设置 `correlated` 消息发送到 broker 会有回调，发消息带一个 `CorrelationData`，回调带回来，用于区分消息`publisher-returns` 参数，配合 `mandatory` 可以在消息投递队列的时候有回调，能保证消息到达队列

然后我们在 `MQConfig`里设置一下回调监听，实际开发可以根据业务逻辑做相应处理：

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (ack) {
        log.info("消息发送到broker: " + correlationData);
    } else {
        log.error("消息发送失败:" + cause);
    }
});
rabbitTemplate.setReturnsCallback(returned -> log.info("消息\"{}\"发送到队列失败:{}",
        new String(returned.getMessage().getBody()),
        returned.getReplyText()));
```

我们再增加一个接口来测试一下回调：

```java
@GetMapping("/direct-confrim")
public String directConfirm() {

    CorrelationData data = new CorrelationData();
    log.info("准备发送消息1-id: " + data.getId());
    rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, MQConsts.QUEUE1, new MsgBean("direct message1"), data);
    ThreadUtil.sleep(1000);
    log.info("----------------------------------------------");
    data = new CorrelationData();
    log.info("准备发送消息2-id: " + data.getId());
    rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "abcde", new MsgBean("direct message2"), data);
    return "OK";
}
```

跑一下测试：

![image-20210127145816665](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127145816665.png)

发现消息1、2都成功发送到了MQ，但是2由于 routingkey:abcde 匹配不到队列，发送失败，回调：NO_ROUTE

### 2. MQ可靠性

MQ 可靠性就是保证消息在 MQ内部不丢失，有一下三点考量：

1. Exchange 持久化，查看源码可知：我们使用单参数构造器创建 Exchange 默认就是 durable 的

![image-20210127155315873](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127155315873.png)

2. Queue 持久化，一样，spring-amqp 给我们默认就是 durable的

![image-20210127155749822](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127155749822.png)

3. 消息持久化，这个需要设置消息的 delivery_mode 为 2，这个也是默认值

我们使用的 `convertAndSend`方法内部调用了 `convertMessageIfNecessary`方法，构造 Message，传入了 `new MessageProperties()`

`MessageProperties`中有静态代码块：

```java
static {
    DEFAULT_DELIVERY_MODE = MessageDeliveryMode.PERSISTENT;
    DEFAULT_PRIORITY = 0;
}
```

既然默认就是持久化的，测试一下，声明一个 Exchange 绑定一个队列，但是不声明消费者：

```java
/*
 * 持久化测试
 */
FanoutExchange exchange = new FanoutExchange(MQConsts.DURABLE_EXCHANGE);
Queue queue = new Queue(MQConsts.DURABLE_QUEUE);
amqpAdmin.declareExchange(exchange);
amqpAdmin.declareQueue(queue);
amqpAdmin.declareBinding(BindingBuilder.bind(queue).to(exchange));
```

这次我们通过UI界面来发一下消息， 点击我们声明的新队列 Queue4，注意 Delivery mode 选 2，点击 Publish message

![image-20210127163622148](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127163622148.png)

可以看到这个队列里有一条消息留在那里：

![image-20210127163830986](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127163830986.png)

我们重启一下 rabbitmq：

![image-20210127164002818](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127164002818.png)

刷新一下页面，可以看到由于刚启动，Message rates 都是空的，但是 queue4 的消息还在：

![image-20210127164132426](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127164132426.png)

### 3. 消费者可靠性

消费者也可能丢失消息，比如我取了消息，还没处理，服务挂了，这样消息就丢了。

解决的方法是关闭 RabbitMQ 的自动 `ack`，我们处理完消息之后手动 `ack`，这样就可以确保消息消费后才从消息队列移除。

开启手动 ack，消费端 yml 里配置：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 开启消费端消息手动ack
        acknowledge-mode: manual
```

这时候，我们的接收的处理就要改写一下了， 我们改写一下 queue3 的消费者：

```java
@RabbitListener(queues = MQConsts.QUEUE3)
@RabbitHandler
public void receiveQueue3ManualAck(MsgBean msgBean, Message message, Channel channel) {
    // deliveryTag 在通道内顺序自增
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    try {
        // 消费消息
        log.info("queue3 receive msg -> :" + JSONUtil.toJsonStr(msgBean));
        // 通知MQ已经成功消费,可以ack, false 表示不批量
        channel.basicAck(deliveryTag, false);
    } catch (Exception e) {
        e.printStackTrace();
        try {
            // 处理失败,重新放回MQ
            channel.basicRecover();
        } catch (IOException ioException) {
            ioException.printStackTrace();
        }
    }
}
```

再往 queue3 发一条消息：

```java
@GetMapping("/ack")
public String consumerMaualAck() {
    rabbitTemplate.convertAndSend(MQConsts.DIRECT_EXCHANGE, "queue3", new MsgBean("manual ack"));
    return "OK";
}
```

![image-20210127171638141](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210127171638141.png)

可以看到顺利接受了消息（null 是我这条消息没有带`CorrelationData`）

### 4. MQ高可用

上面完成了消息从生产者到消费者的可靠性投递，但是由于我们的 MQ 是单节点，挂了还是不可用，实际生产中，会部署 RabbitMQ 集群，这里注意，RabbitMQ 的集群只能提高**吞吐量**，并不能高可用。想要做到高可用，可以在集群中对有需要的队列配置**镜像队列**，真正实现高可用，可以在 UI 界面的 Admin Policies 里配置，可以参考[官方文档](https://www.rabbitmq.com/ha.html)

## 6. 延迟队列

消息队列还有一个比较常见的使用场景：延迟队列，比如订单1小时不支付，自动取消，返还库存等

RabbitMQ 实现延迟队列有两种方式：死信队列 和 延迟队列插件

死信队列结合消息超时时间 TTL 可以实现延迟队列效果，但是实现起来没有用插件来的方便，这里我们介绍一下插件使用

插件的[下载地址](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases)，我使用的是最新的 3.8.9 版本，下载插件，拷贝到容器内，并启动：

```bash
# 拷贝插件到容器内
docker cp ~/Downloads/rabbitmq_delayed_message_exchange-3.8.9-0199d11c.ez rabbitmq:/plugins
# 进入容器内
docker exec -it rabbitmq /bin/bash
cd /plugins
ls |grep delayed
# 启动插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
# 退出容器
exit
# 重启MQ
docker restart rabbitmq
```

我们代码里声明一下延迟队列和它的 Exchange：

```java
// MQConsts.java
String DELAYED_QUEUE = "delayed.queue";
String DELAYED_ECCHANGE = "delayed.exchange";

// MQConfig
Queue delayedQueue = new Queue(MQConsts.DELAYED_QUEUE);
// 根据官方文档,我们这样声明一个Exchange
Map<String, Object> args = MapUtil.newHashMap();
args.put("x-delayed-type", "direct");
CustomExchange customExchange = new CustomExchange(MQConsts.DELAYED_ECCHANGE, "x-delayed-message", true, false, args);
amqpAdmin.declareQueue(delayedQueue);
amqpAdmin.declareExchange(customExchange);
// 这里路由键我们就用延迟队列的名字
amqpAdmin.declareBinding(BindingBuilder.bind(delayedQueue).to(customExchange).with(MQConsts.DELAYED_QUEUE).noargs());
```

发送延迟消息这样发：

```java
@GetMapping("/delay")
public String delayQueue() {
    log.info("send time:{}", System.currentTimeMillis());
    rabbitTemplate.convertAndSend(MQConsts.DELAYED_ECCHANGE, MQConsts.DELAYED_QUEUE, new MsgBean("delay queue"), message -> {
        message.getMessageProperties().setHeader("x-delay", 5000);
        return message;
    });
    return "OK";
}
```

我们先把消费端手动 ack 关了，方便测试：

```yaml
listener:
  simple:
    acknowledge-mode: auto
```

消费者代码：

```java
@RabbitListener(queues = MQConsts.DELAYED_QUEUE)
@RabbitHandler
public void receiveDelayedQueue(MsgBean msgBean) {
    log.info("delayed queue receive msg -> :" + JSONUtil.toJsonStr(msgBean));
    log.info("receive time:{}", System.currentTimeMillis());
}
```

测试一下：

![image-20210128092946695](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20210128092946695.png)

可以看到，实现了延迟5秒发送的效果，30 发送 35 收到。

但是延迟队列发的瞬间会提示发到队列失败，因为它是到了时间才往队列发，而生产者不会等那么久，所以这里如果做消息处理要注意一下。

本篇到此结束，已经写得很长了，太长会影响阅读性。

源码及脚本都在[Github](https://github.com/RigerWu/web-starter-demos)上

Enjoy it!

### 参考资料

- [1] [RabbitMQ实战指南](https://book.douban.com/subject/27591386/)
- [2] [RabbitMQ官方文档](https://www.rabbitmq.com/documentation.html)
- [3] [spring-amqp文档](https://docs.spring.io/spring-amqp/docs/current/reference/html/)
