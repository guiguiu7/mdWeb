### 简述

> 生产者和消费者与RabbitMQ Broker建立```TCP```连接。每个应用程序通常维护一个TCP连接，在该连接上可以创建多个```Channel```用于不同的操作。
>
> 生产者通过Channel将消息发送到Exchange。```Exchange```根据绑定规则和路由键决定是否将消息路由到```Queue```。如果消息无法路由到任何Queue，会触发```ReturnsCallback```。
>
> 消费者通过Channel订阅Queue。当消息到达Queue时，RabbitMQ通过Channel将消息推送给消费者。消费者处理完成后通过Channel发送确认(ACK/NACK)。

### 安装RabbitMQ

```
docker run -d --name rabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin@2025 -p 5672:5672 -p 15672:15672 rabbitmq:4-management

```

### 引入依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 配置文件

```properties
# RabbitMQ 配置
spring.rabbitmq.host=103.210.237.3
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin@2025

# 用于消息队列可靠投递
spring.rabbitmq.publisher-confirm-type=correlated
spring.rabbitmq.publisher-returns=true
# 手动ACK确认
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

### Java代码实现

使用RabbitMQ进行收发信息需要先创建***Exchange***、***Queue***并且通过***Binding***将两者进行绑定。

```java
@Configuration
public class MyRabbitConfig {

    @Autowired
    RabbitTemplate rabbitTemplate;

    @Autowired
    AmqpAdmin amqpAdmin;

    private static final Logger logger = LogManager.getLogger(MyRabbitConfig.class);
    @PostConstruct
    public void init() {
     	// 创建Exchange、Queue、Binding
        amqpAdmin.declareExchange(new DirectExchange("my.direct.exchange"));
        amqpAdmin.declareQueue(new Queue("my.rabbitmq.queue"));
        amqpAdmin.declareBinding(new Binding("my.rabbitmq.queue",
                Binding.DestinationType.QUEUE,
                "my.direct.exchange",
                "java.rabbitmq",
                null));
        // 消息发送到exchange时回调
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            logger.log(Level.WARN,"消息发送确认回调：{},ACK==>{},原因==>{}",correlationData,ack,cause);
        });
        // 消息发送到queue失败时回调
        rabbitTemplate.setReturnsCallback(returns -> {
            logger.log(Level.WARN,"消息发送失败回调：{},code==>{},原因==>{},交换机==>{},路由键==>{}",
                    new String(returns.getMessage().getBody()),
                    returns.getReplyCode(),
                    returns.getReplyText(),
                    returns.getExchange(),
                    returns.getRoutingKey());
        });
    }
}
```

***producer***需要使用***RabbitTemplate***的convertAndSend方法发送消息，其中需要指定交换机、路由键、信息

```java
@RestController
public class SendRabbitTest {

    @Autowired
    RabbitTemplate rabbitTemplate;

    @PostMapping("/sendRabbitTest")
    public String sendRabbitTest(@RequestParam(value = "num",defaultValue = "10") Integer num) {
        // 这里可以添加发送 RabbitMQ 消息的逻辑
        for (int i = 0; i < num; i++) {
            if (i % 2 == 0) {
                // 模拟发送字符串消息
                rabbitTemplate.convertAndSend("my.direct.exchange",
                        "java.rabbitmq",
                        "Sending String message " + (i + 1),
                        new CorrelationData(new Date().toString()));
            } else {
                // 模拟发送对象
                RabbitTest rabbitTest = new RabbitTest();
                rabbitTest.setMessage("Sending RabbitTest message " + (i + 1));
                rabbitTemplate.convertAndSend("my.direct.exchange",
                        "java.rabbitmq",
                        rabbitTest,
                        new CorrelationData(new Date().toString()));
            }
        }
        return "RabbitMQ test message sent successfully!";
    }
}
```

***consumer***需要添加**@RabbitListener(queues = "my.rabbitmq.queue") **对队列进行监听，配合```@RabbitHandler```注解会将不同消息类型分发到匹配的处理方法。当***@RabbitHandler(isDefault = true)***用于接收数据类型不匹配时进行的处理方法。

使用***Channel***通过***basicNack***和***basicAck***两个方法指定***deliveryTag***手动拒绝消息和接收消息（如果拒绝的消息可以重新入队，但是重新入队后自身的deliveryTag会+1，其后的所有deliveryTag也会+1）

```java
@Service
@RabbitListener(queues = "my.rabbitmq.queue") // 替换为实际的队列名称
public class ReceiveRabbitServiceImpl implements ReceiveRabbitService {
    // 消费者确认
    // 默认自动确认，消息就会删除，但服务器宕机会导致消息丢失
    // 手动确认，只有当我们发送确认信号给RabbitMQ，才会删除消息
    @RabbitHandler
    public void receiveMessage(@Payload String message,
                               Channel channel,
                               @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag){
        // 消息接收手动确认
        try {
            if (deliveryTag % 2 != 0) {
                // 拒绝并重新入队
                channel.basicNack(deliveryTag, false, true);
            }else {
                // 确认并且删除队列中消息
                channel.basicAck(deliveryTag, false);
            }
        } catch (IOException e) {
            System.out.println("Failed to acknowledge message: " + e.getMessage());
        }
    }
    // 处理接收到的 RabbitTest 对象消息 
    @RabbitHandler
    public void receiveMessage(RabbitTest rabbitTest,
                               Channel channel,
                               @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag)
```

##### 项目启动类上需要@EnableRabbit开启服务

#### ==注意事项==

需在 Spring 中配置消息转换器（如 JSON 转换器），否则无法正确反序列化：

```java
@Bean
public MessageConverter jsonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}
```



### TTL与死信队列实现延迟队列

TTL 是 RabbitMQ 中一个消息或者队列的属性，表明**一条消息或者该队列中的所有消息的最大存活时间**，单位是毫秒。换句话说，如果一条消息设置了 TTL 属性或者进入了设置 TTL 属性的队列，那么这条消息**如果在 TTL 设置的时间内没有被消费，则会成为“死信”**。如果同时配置了队列的 TTL 和消息的 TTL，那么**较小的那个值**将会被使用。

消息过期变成死信后交给死信交换机处理

##### 两种设置TTL的方法

1.创建队列并设置x-message-ttl

```java
@Bean
public Queue delayQueue() {
    String queueName = "delay_queue";
    Map<String, Object> args = new HashMap<>(1);
    args.put("x-message-ttl", "6000");
    return new Queue(queueName, true, false, false, args);
}
```

2.针对每条信息设置TTL

```java
rabbitTemplate.convertAndSend(exchange, routingKey, (message) -> {
 message.getMessageProperties().setExpiration("6000");
 return message;
});
```

区别：

- 设置了队列的 TTL 属性，那么一旦消息过期，就会被队列丢弃
- 给消息设置 TTL 属性，消息过期也不一定会马上丢弃，因为消息是否过期是在即将**投递到消费者之前判定的**，如果队列存在消息积压问题，那么已过期的消息可能还会存活较长些时间



#### 队列 TTL 属性来实现延迟队列

##### 声明交换机、队列

```java
// 配置延迟队列
@Bean
public TopicExchange delayExchange() {
    String exchangeName = "delay_exchange";
    return new TopicExchange(exchangeName);
}
@Bean
public Queue delayQueueA() {
    String queueName = "delay_queue_A";
    // 设置死信发送至 dlx_exchange 交换机，设置路由键为 bind.dlx.A
    String dlxExchangeName = "dlx_exchange";
    String bindDlxRoutingKeyA = "bind.dlx.A";
 
    Map<String, Object> args = new HashMap<>(3);
    // 设置队列的延迟属性，6秒
    args.put("x-message-ttl", 6000);
    args.put("x-dead-letter-exchange", dlxExchangeName);
    args.put("x-dead-letter-routing-key", bindDlxRoutingKeyA);
    return new Queue(queueName, true, false, false, args);
}
@Bean
public Binding bindingDelayExchange() {
    String routingKey = "bind.delay.A";
    return BindingBuilder.bind(delayQueueA()).to(delayExchange()).with(routingKey);
}
 
// 配置死信队列
@Bean
public TopicExchange dlxExchange() {
    String exchangeName = "dlx_exchange";
    return new TopicExchange(exchangeName);
}
@Bean
public Queue dlxQueueA() {
    String queueName = "dlx_queue_A";
    return new Queue(queueName);
}
@Bean
public Binding bindingDlxExchange() {
    String routingKey = "#.A";
    return BindingBuilder.bind(dlxQueueA()).to(dlxExchange()).with(routingKey);
}
```

##### yaml配置

```yaml
spring:
  rabbitmq:
    host: 192.168.159.129
    port: 5672
    username: admin
    password: admin
    # 虚拟host 可以不设置,使用 server 默认 host
    virtual-host:
    listener:
      simple:
        default-requeue-rejected:
        acknowledge-mode: manual
```

##### 生产者

```java
@Test
public void demo_10_Producer() {
    String exchange = "delay_exchange_A";
    String routingKey = "delay.routing.key.A";
    String msg = "发送给延迟队列 delay_queue_A 的消息";
    System.out.println("当前时间：" + simpleDateFormat.format(new Date()) + "开始发送消息：" + msg);
    rabbitTemplate.convertAndSend(exchange, routingKey, msg);
}
```

##### 缺陷

如果这样使用的话，每增加一个新的时间需求，就要新增一个队列。



#### 消息的 TTL 属性来实现延迟队列

##### 声名交换机、队列

```java
// 配置延迟队列
@Bean
public TopicExchange delayExchange() {
    String exchangeName = "delay_exchange";
    return new TopicExchange(exchangeName);
}
@Bean
public Queue delayQueueB() {
    String queueName = "delay_queue_B";
    // 设置死信发送至 dlx_exchange 交换机，设置路由键为 bind_dlx_B
    String dlxExchangeName = "dlx_exchange";
    String bindDlxRoutingKeyB = "bind.dlx.B";
 
    Map<String, Object> args = new HashMap<>(2);
    args.put("x-dead-letter-exchange", dlxExchangeName);
    args.put("x-dead-letter-routing-key", bindDlxRoutingKeyB);
    return new Queue(queueName, true, false, false, args);
}
@Bean
public Binding bindingDelayExchange() {
    String routingKey = "bind.delay.B";
    return BindingBuilder.bind(delayQueueB()).to(delayExchange()).with(routingKey);
}
 
// 配置死信队列
@Bean
public TopicExchange dlxExchange() {
    String exchangeName = "dlx_exchange";
    return new TopicExchange(exchangeName);
}
@Bean
public Queue dlxQueueB() {
    String queueName = "dlx_queue_B";
    return new Queue(queueName);
}
@Bean
public Binding bindingDlxExchange() {
    String routingKey = "#.B";
    return BindingBuilder.bind(dlxQueueB()).to(dlxExchange()).with(routingKey);
}
```

##### 生产者

```java
@Test
public void producer_B() throws Exception {
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    String exchange = "delay_exchange";
    String routingKey = "bind.delay.B";
    String msg = "我是第一条消息";
    // 延迟时间
    String delayTime = "6000";
 
    System.out.println("当前时间：" + simpleDateFormat.format(new Date()) + "开始发送消息：" + msg + "  延迟的时间为：" + delayTime);
    rabbitTemplate.convertAndSend(exchange, routingKey, msg, new MyMessagePostProcessor(delayTime));
 
    Thread.sleep(30000L);
}
```

##### 消费者

```java
@RabbitListener(queues = "dlx_queue_B")
public void receiverB(@Payload String msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
    System.out.println("当前时间：" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + " 死信队列 - dlx_queue_B 收到消息：" + msg);
    channel.basicAck(deliveryTag, false);
}
```

```java
/**
 * 因为要给消息设置 TTL，这里创建了一个 MessagePostProcessor 的实例来设置过期时间
 */
class MyMessagePostProcessor implements MessagePostProcessor {
    // 延迟时间 毫秒
    private String delayTime;
 
    MyMessagePostProcessor(String delayTime) {
        this.delayTime = delayTime;
    }
 
    @Override
    public Message postProcessMessage(Message message) throws AmqpException {
        // 设置延迟时间
        message.getMessageProperties().setExpiration(delayTime);
        return message;
    }
}
```

##### 缺陷

但是上面我们也提到了消息过期也不一定会马上丢弃。消息到了过期时间可能并不会按时“死亡“，因为 RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，索引如果第一个消息的延时时长很长，而第二个消息的延时时长很短，则第二个消息并不会优先得到执行。

==延迟交换机和死信交换机可以复用同一个交换机，通过路由键的将消息发送到延迟队列或者是死信队列==



##### 创建队列的两种方式

###### 1.使用AmqpAdmin手动声名队列

```java
amqpAdmin.declareQueue(new Queue("my.rabbitmq.queue"));
```

###### 2.直接使用@Bean创建对象

```java
@Bean
public Queue delayQueue() {
    String queueName = "delay_queue";
    Map<String, Object> args = new HashMap<>(1);
    args.put("x-message-ttl", "6000"); // 设置消息过期时间（毫秒）
    return new Queue(queueName, true, false, false, args);
}
```

1. **自动声明机制**
   Spring AMQP 会自动检测所有类型为 `Queue`、`Exchange` 或 `Binding` 的 `@Bean`，并在应用启动时通过内部的 `AmqpAdmin` 实例自动声明它们。
2. **容器管理**
   当 `RabbitAutoConfiguration` 生效时，它会自动创建 `CachingConnectionFactory`、`RabbitTemplate` 和 `RabbitAdmin`（`AmqpAdmin` 的实现类）。这些组件会自动处理队列声明。
3. **声明时机**
   队列会在以下时机自动声明：
   - 应用启动时
   - 第一次使用 `RabbitTemplate` 发送消息到该队列时
   - 消费者订阅该队列时



### 消息可靠性

##### 1.重复消费

保证消息不被重复消费，需要业务消费接口设计应该具有**幂等性**。（防重表、消息中redelivered字段判断是否重新投递）

#####  2.消息丢失

可以通过消息确认机制来保证消息正常的传递（生产者、消费者）如**手动ACK**、**confirm回调**。消息记录可以在数据库中进行保存，定期将失败的消息重新发送。

##### 3.消息积压

为了防止消息积压导致MQ服务宕机，可以将消息取出保存到消息记录数据库中，后续慢慢进行处理。
