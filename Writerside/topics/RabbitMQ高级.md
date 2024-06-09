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