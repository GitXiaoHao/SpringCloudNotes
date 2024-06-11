# RabbitMQ高级

## 发送者可靠性
### 发送者重连
- 有的时候由于网络波动，可能会出现发送者连接MQ失败的情况。通过配置我们可以开启连接失败后的重连机制
```yaml
spring:
  rabbitmq:
    connection-timeout: 1s # 设置MQ的连接超时时间
    template:
      retry:
        enabled: true # 开启超时重试机制
        initial-interval: 1000ms # 失败后的初始等待时间
        multiplier: 1 # 失败后下次的等待时长倍数
        max-attempts: 3 # 最大重试次数
```
- 注意:
  当网络不稳定的时候，利用重试机制可以有效提高消息发送的成功率。不过 SpringAMOP 提供的重试机制
  是阻塞式的重试，也就是说多次重试等待的过程中，当前线程是被阻塞的，会影响业务性能。
  如果对于业务性能有要求，建议禁用重试机制。如果一定要使用，请合理配置等待时长和重试次数，当然也
  可以考虑使用异步线程来执行发送消息的代码。
### 发送者确认
- SpringAMQP提供了Publisher Confirm和Publish Return两种确认机制。开启确认机制后，当发送者发送消息给MQ后，MQ会返回确定结果给发送者。返回的结果有以下情况
  - 消息投递到了MQ，但是路由失败。此时会通过PublisherReturn返回路由异常原因，然后返回ACK，告知投递成功
  - 临时消息投递到了MQ，并且入队成功，返回ACK，告知投递成功
  - 持久消息投递到了MQ，并且入队完成持久化，返回ACK，告知投递成功
  - 其他情况都会返回NACK，告知投递失败
![mq4.png](mq4.png)
#### SpringAMQP实现发送者确认
- 在publisher这个微服务的application.yaml中添加配置
```yaml
    publisher-confirm-type: correlated
    publisher-returns: true 
```
  - 这里publisher-confirm-type有三种模式
    - none：关闭confirm机制
    - simple：同步阻塞等待MQ的回执消息
    - correlated：MQ异步回调方式返回回执消息
- 每个RabbitTemplate只能配置一个ReturnCallback，因此需要在项目启动过程中配置
```Java

import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Configuration;

/**
 * @author yuhao
 * @time 2024/6/10 0:49
 **/
@Slf4j
@Configuration
@RequiredArgsConstructor
public class MqConfig {
    private final RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        rabbitTemplate.setReturnsCallback(returnedMessage -> {
            log.error("callback");
            log.debug("交换机：{}", returnedMessage.getExchange());
            log.debug("routingKey：{}", returnedMessage.getRoutingKey());
            log.debug("message：{}", returnedMessage.getMessage());
            log.debug("replyCode：{}", returnedMessage.getReplyCode());
            log.debug("replyText：{}", returnedMessage.getReplyText());
        });
    }
}

```
- 发送消息，指定消息ID，消息ConfirmCallback
```Java
correlationData.getFuture().addCallback(
                confirm -> {
                    if (confirm.isAck()) {
                        //消息顺利给了交换机
                        System.out.println("发送成功ACK，confirm = " + confirm + ",  id:" + correlationData.getId());
                    } else {
                        //消息给交换机失败
                        System.out.println("发送失败NACK，confirm = " + confirm + ",  id:" + correlationData.getId());
                    }
                },
                throwable -> {
                    //发生错误，链接mq异常，mq未打开等...报错回调
                    System.out.println("发送失败throwable = " + throwable + ",  id:" + correlationData.getId());
                }
                
```
#### 提示
- 老版本的spring-amqp在CorrelationData上设置ConfirmCallback
- 新版本ConfirmCallback和ReturnsCallback都需要在RabbitTemplate中设置，同时ConfirmCallback中默认无法得到消息内容
##### RabbitTemplate
```Java
	@Bean
    public RabbitTemplate rabbitTemplate(final ConnectionFactory connectionFactory) {
        final RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        //设置confirm callback
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            String body = "1";
            if (correlationData instanceof EnhancedCorrelationData) {
                body = ((EnhancedCorrelationData) correlationData).getBody();
            }
            if (ack) {
                //消息投递到exchange
                log.debug("消息发送到exchange成功:correlationData={},message_id={} ", correlationData, body);
                System.out.println("消息发送到exchange成功:correlationData={},message_id={}"+correlationData+body);
            } else {
                log.debug("消息发送到exchange失败:cause={},message_id={}",cause, body);
                System.out.println("消息发送到exchange失败:cause={},message_id={}"+cause+body);
            }
        });
        
        //设置return callback
        rabbitTemplate.setReturnsCallback(returned -> {
            Message message = returned.getMessage();
            int replyCode = returned.getReplyCode();
            String replyText = returned.getReplyText();
            String exchange = returned.getExchange();
            String routingKey = returned.getRoutingKey();
            // 投递失败，记录日志
            log.error("消息发送失败，应答码{}，原因{}，交换机{}，路由键{},消息{}",
                    replyCode, replyText, exchange, routingKey, message.toString());
        });
        return rabbitTemplate;
    }

```
##### EnhancedCorrelationData
- 原始的CorrelationData，目前已经无法从中获取消息内容，也就是说现在的ConfirmCallback无法获取到消息的内容，因为设计上只关注是否投递到exchange成功。如果需要在ConfirmCallback中获取消息的内容，需要扩展这个类，并在发消息的时候，放入自定义数据。
```Java
public class EnhancedCorrelationData extends CorrelationData {
    private final String body;

    public EnhancedCorrelationData(String id, String body) {
        super(id);
        this.body = body;
    }

    public String getBody() {
        return body;
    }
}


```
##### 发送消息
在EnhancedCorrelationData把消息本身放进去，或者如果你有表记录消息，你可以只放入其id。这样触发ConfirmCallback的时候，就可以获取消息内容。
```Java
		public void notifyPayResult() {
		String message = "TEST Message";
        Message message1 = MessageBuilder.withBody(message.getBytes(StandardCharsets.UTF_8))
                .setDeliveryMode(MessageDeliveryMode.PERSISTENT)
                .build();
        CorrelationData correlationData = new EnhancedCorrelationData(UUID.randomUUID().toString(), message.toString());
        rabbitTemplate.convertAndSend(PayNotifyConfig.PAYNOTIFY_EXCHANGE_FANOUT,"", message1, correlationData);
    }

```
## 数据持久化
- 在默认情况下，RabbitMQ会将接收到的信息保存在内存中以降低消息收发的延迟。这样会导致两个问题
  - 一旦MQ宕机，内存中的消息会丢失
  - 内存空间有限，有消费故障或处理过慢时，会导致消息挤压，引发MQ阻塞
- RabbitMQ实现数据持久化包括三个方面
  - 交换机持久化 默认开启
  - 队列持久化 默认开启
  - 消息持久化 默认开启
    - 设置非持久化的
      ```Java
        Message message1 = MessageBuilder.withBody(message.getBytes(StandardCharsets.UTF_8))
                .setDeliveryMode(MessageDeliveryMode.NON_PERSISTENT)
                .build();
       ```
## Lazy Queue
- 从RabbitMQ的3.6.0版本开始，就增加了LazyQueue的概念，也就是惰性队列
- 惰性队列的特征如下
  - 接收到消息后直接存入磁盘，不在存储到内存
  - 消费者要消费消息时才会从磁盘读取并加载到内存（可以提前缓存部分消息到内存，最多2048条）
- 在3.12版本后，所有队列都是LazyQueue模式，无法更改
![lazyQueue1.png](lazyQueue1.png)
## 消费者可靠性 - 消费者确认机制
- 消费者确认机制是为了确认消费者是否成功处理消息。当消费者处理消息结束后，应该向RabbitMQ发送一个回执，告知RabbitMQ自己消息处理状态
  - ack：成功处理消息，RabbitMQ从队列中删除该消息
  - nack：消息处理失败，RabbitMQ需要再次投递消息
  - reject：消息处理失败并拒绝该消息，RabbitMQ从队列中删除该消息
- SpringAMQP已经实现了消息确认机制功能。并允许通过配置文件选择ACK处理方式
  - none：不处理。即消息投递给消费者后立刻ack，消息会立刻从MQ删除。非常不安全，不建议使用
  - manual：手动模式。需要自己在业务代码中调用api，发送ack或reject，存在业务入侵，但更灵活
  - auto：自动模式。SpringAMQP利用AOP对消息处理逻辑做环绕增强，当业务正常执行时则自动返回ack
    - 当业务出现异常时，根据异常判断返回不同结果
      - 如果业务异常，会自动返回nack
      - 如果是消息处理或校验异常，自动返回reject
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual 
```
## 消费者可靠性 - 消费者失败重试策略
- SpringAMQP提供了消费者失败重试机制，在消费者出现异常时利用本地重试，而不是无限的requeue到mq。我们可以通过在配置文件中添加配置来开启重试机制
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1
        retry:
          enabled: true # 开启超时重试机制
          initial-interval: 1000ms # 失败后的初始等待时间
          multiplier: 1 # 失败后下次的等待时长倍数
          max-attempts: 3 # 最大重试次数
          stateless: true # true无状态 如果业务中包含事务，则改为false 
```
- 开启重试模式后，重试次数耗尽，如果消息依然失败，则需要有MessageRecoverer接口来处理，它包含三种不同的实现
  - RejectAndDontRequeueRecoverer：重试耗尽后，直接reject，丢弃消息。默认就是这种方式
  - ImmediateRequeueMessageRecoverer：重试耗尽后，返回nack，消息重新入队
  - RepublishMessageRecoverer：重试耗尽后，将失败消息投递到指定的交换机
### 失败消息处理策略
- 将失败处理策略改为RepublishMessageRecoverer
  - 首先定义接受失败消息的交换机、队列及其绑定关系
  - 然后定义RepublishMessageRecoverer
```Java

import lombok.RequiredArgsConstructor;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.retry.MessageRecoverer;
import org.springframework.amqp.rabbit.retry.RepublishMessageRecoverer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author yuhao
 * @time 2024/6/10 16:19
 **/
@Configuration
@RequiredArgsConstructor
public class ErrorMessageConfiguration {
    private final RabbitTemplate rabbitTemplate;
    @Bean
    public DirectExchange errorExchange(){
        return ExchangeBuilder.directExchange("error.direct").build();
    }
    @Bean
    public Queue errorQueue(){
        return QueueBuilder.durable("error.queue").build();
    }
    @Bean
    public Binding errorQueueBinding(Queue errorQueue,DirectExchange errorExchange){
        return BindingBuilder.bind(errorQueue).to(errorExchange).with("error");
    }
    @Bean
    public MessageRecoverer messageRecoverer(){
        return new RepublishMessageRecoverer(rabbitTemplate,"error.direct","error");
    }
}
 
 ```
## 消费者可靠性 - 业务幂等性
- 指同一个业务，执行一次或多次对业务状态的影响是一致的
### 唯一消息id
- 方案一，是给每个消息都设置一个唯一id，利用id区分是否是重复消息
  - 每一条消息都生成一个唯一的id，与消息一起投递给消费者
  - 消费者接受到消息后处理自己的业务，业务处理成功后将消息id保存到数据库
  - 如果下次又收到相同消息，去数据库查询判断是否存在，存在则为重复消息放弃处理
#### 发送者
- 只需要打开开关即可自动创建id
```Java
@Bean
    public MessageConverter messageConverter(){
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        
        converter.setCreateMessageIds(true);
        return converter;
    }
```
#### 消费者
```Java
@RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "direct.queue3"),
            exchange = @Exchange(name = "hmall.direct",type = ExchangeTypes.DIRECT),
            key = {"red","blue"}
    ))
    public void listenerDirectQueue3(Message message){
        System.err.println("消费者3：" + Arrays.toString(message.getBody()) + ", " + LocalTime.now());
        System.err.println("消费者3 ID：" + message.getMessageProperties().getMessageId() + ", " + LocalTime.now());
    }
```
## 延迟消息
- 延迟消息：发送者发送消息时指定一个时间，消费者不会立刻收到消息，而是在指定时间之后才收到消息。
- 延迟任务：设置在一定时间之后才执行的任务
### 死信交换机
- 当一个队列中的消息满足下列情况之一时，就会成为死信（dead letter）
  - 消费者使用basic.reject 或 basic.nack声明消费失败，并且消息的requeue参数设置为false
  - 消息是一个过期消息（达到了队列或消息本身设置的过期时间），超时无人消费
  - 要投递的队列消息队积满了，最早的消息可能成为死信
- 如果队列通过dead-letter-exchange属性指定一个交换机，那么该队列中的死信就会投递到这个交换机中。这个交换机称为死信交换机（DLX）
![dlx1.png](dlx1.png)
```Java
@Bean
    public Queue errorQueue(){
        return QueueBuilder.durable("error.queue").deadLetterExchange("dlx.direct").build();
    } 
```
- 设置消息过期时间
```Java
rabbitTemplate.convertAndSend(exchangeName, "", message, new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                message.getMessageProperties().setExpiration("10000");
                return message;
            }
        });
```
### 延迟消息插件
- 这个插件可以将普通交换机改造为支持延迟消息功能的交换机，当消息投递到交换机后可以暂存一定时间，到期后再投递到队列
![delayMessage1.png](delayMessage1.png)
- 插件下载  https://www.rabbitmq.com/community-plugins.html
#### linux下载
- 下载rabbitmq_delayed_message_exchange 插件，然后解压放置到 RabbitMQ 的插件目录。
  /usr/lib/rabbitmq/lib/rabbitmq_server-3.13.3/plugins
` cp rabbitmq_delayed_message_exchange-3.8.0.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.8.8/plugins`
- 进入 RabbitMQ 的安装目录下的 plgins 目录，执行下面命令让该插件生效，然后重启 RabbitMQ
  - `rabbitmq-plugins enable rabbitmq_delayed_message_exchange`
- 插件安装成功，重启MQ `systemctl restart rabbitmq-server`
#### docker 安装
- 先查看RabbitMQ的插件目录对应的数据卷 `docker volume inspect mq-plugins`
- 上传插件到该目录
- 执行命令 安装插件 `docker exec -it mq rabbit-plugins enable rabbitmq_delayed_message_exchange`
- 
#### windows 下载
- 把 下载下来的文件拷贝到RabbitMQ安装目录下的 plugins 目录。
- 进入RabbitMQ安装目录下的 sbin目录，在cmd窗口下执行如下命令使插件生效 如果后面发现在未失效请重启服务再查看
- `rabbitmq-plugins enable rabbitmq_delayed_message_exchange`
-  打开rabbitmq控制台，点击exchange，如果Add a new exchange功能里的Type下拉框里出现x-delayed-message类型，则说明安装成功，可以发布延时消息了。
#### 使用插件
```Java
@RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "delay.queue"),
            exchange = @Exchange(name = "delay.direct",delayed = "true"),
            key = {"hi"}
    ))
    public void listenerDelayQueue(Message message){
        System.err.println("消费者：" + Arrays.toString(message.getBody()) + ", " + LocalTime.now());
        System.err.println("消费者 ID：" + message.getMessageProperties().getMessageId() + ", " + LocalTime.now());
    }
```

![delayMessage2.png](delayMessage2.png)
- 发送消息时需要通过消息头x-delay来设置过期时间
![delayMessage3.png](delayMessage3.png)
```Java
@Test
public void testDelayQueue(){
    String exchangeName = "delay.direct";
    String message = "hello,delay";
    rabbitTemplate.convertAndSend(exchangeName, "hi", message, message1 -> {
        message1.getMessageProperties().setDelay(10000);
        return message1;
    });
}
```
