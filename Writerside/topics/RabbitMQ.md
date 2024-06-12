# RabbitMQ
## 同步调用
- 扩展性差
- 性能下降
- 级联失败
## 异步调用
- 异步调用通常是基于消息通知的方式 包含三个角色
  - 消息发送者；投递消息的人，就是原来的调用者
  - 消息接收者：接收和处理消息的人，就是原来的服务提供者
  - 消息代理：管理、暂存、转发消息，可以理解成服务器
### 优点
- 解除耦合，扩展性强
- 无需等待，性能好
- 故障隔离
- 缓存消息，流量削峰填谷
### 缺点
- 不能立即得到调用结果，时效性差
- 不确定下游业务执行是否成功
- 业务安全依赖于Broker的可靠性
## MQ技术选型
- 消息队列，存放消息队列。也就是异步调用的Broker
![mq1.png](mq1.png)
## RabbitMQ - 安装部署
### docker安装
```Shell
docker run \
-e RABBITMQ_DEFAULT_USER=iyuhao \
-e RABBITMQ_DEFAULT_PASS=yu123456 \
-v mq-plugins:/plugins \
--name mq \
--hostname mq \
-p 15672:15672 \
-p 5672:5672 \
--network hm-net \
-d \
rabbitmq:3.8-management
```
- 如果拉取镜像困难 则利用docker load加载
- 15672：RabbitMQ提供的管理控制台的端口
- 5672：RabbitMQ的消息发送处理接口
### windows安装
- rabbitMQ安装程序下载路径：https://github.com/rabbitmq/rabbitmq-server/releases
- erlang环境安装程序下载路径：https://www.erlang.org/downloads
#### 安装erlang
- 点击刚才下载的otp_win64_23.0.exe。
- 接下来配置环境变量，常规操作，新建系统变量-键入变量名ERLANG_HOME，键入变量值:erlang安装路径
- 然后添加系统path路径中，添加 ： `%ERLANG_HOME%\bin`
- 然后打开cmd，输入erl，看到我们的erlang版本号，就说明安装成功了
#### 安装RabbitMQ {id="rabbitmq_1"}
- 双击我们刚才下载的rabbitmq-server-3.8.5程序，next，install即可，此处需要注意，如果要自定义安装路径的话，路径中最好不要存在中文，会出现错误。
- 安装完成之后，需要我们激活rabbitmq_management
- 打开cmd，进到sbin目录下，运行命令
  - `rabbitmq-plugins enable rabbitmq_management`
- 执行成功之后会看到如下图：三个插件被启动
#### 验证
- 上面的命令执行成功之后，我们就可以通过http://localhost:15672来访问web端的管理界面
- 初始可以通过用户名：guest  密码guest来登录。
#### 其他命令
```Shell
net start RabbitMQ  启动
net stop RabbitMQ  停止
rabbitmqctl status  查看状态
```
## 基本介绍
- virtual-host：虚拟主机，起到数据隔离的作用
- publisher：消息发送者
- consumer：消息的消费者
- queue：队列、存储消息
- exchange：交换机，负责路由消息
![mq2.png](mq2.png)
## Java客户端
### 引入依赖
```xml
 <!--AMQP依赖，包含RabbitMQ-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```
### 创建队列
- 利用mq控制台创建队列simple.queue
### 配置mq服务端信息 {id="mq_1"}
```yaml
spring:
  rabbitmq:
    host: 192.168.139.1
    port: 5672
    virtual-host: /hmall # 主机名
    username: hmall # 用户名
    password: 123 # 密码
    
```
### 发送消息
```Java
@Resource
    private RabbitTemplate rabbitTemplate;
    @Test
    public void testSimpleQueue(){
        String queueName = "simple.queue";
        String message = "hello";
        rabbitTemplate.convertAndSend(queueName,message);
    }
```
### 接收消息
- SpringAMQP提供声明式的消息监听，只需要通过注解在方法上声明要监听的队列名称，就会把消息传递给当前方法
```Java
@Component
@Slf4j
public class SpringRabbitListener {
    @RabbitListener(queues = "simple.queue")
    public void listenerSimpleQueue(String message){
        log.info("监听到消息：{}",message);
    }
}

```
## Work Queues
- 任务模型 让多个消费者绑定到一个队列，共同消费队列中的消息
### 消费者消息推送限制
- 默认情况下，RabbitMQ的会将消息依次轮询投递给绑定在队列上的每一个消费者。但这个并没有考虑到消费者是否已经处理完消息，可能出现消息队列
- 因此需要修改配置，设置preFetch为1，确保同一时刻最多投递给消费者1条消息
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 #每次只能接受一个消息
```
### Work模型使用
- 多个消费者绑定到一个队列，可以加快消息处理速度
- 同一条消息只会被一个消费者处理
- 通过设置prefetch来控制消费者预取的消息数量，处理完一条再处理下一条
## Fanout交换机 - 广播
- 交换机的作用主要是接受发送者发送的消息，并将消息路由到与其绑定的队列
- Fanout Exchange 会将接收到的消息路由到每一个跟其绑定的queue，也叫广播模式
### 使用
#### 声明两个队列
![fanout1.png](fanout1.png)
#### 声明交换机
![fanout2.png](fanout2.png)
#### 交换机和队列绑定
![fanout3.png](fanout3.png)
#### 接收消息
```Java
 @RabbitListener(queues = "fanout.queue1")
    public void listenerFanoutQueue1(String message){
        System.err.println("消费者1：" + message + ", " + LocalTime.now());
    }
    @RabbitListener(queues = "fanout.queue2")
    public void listenerFanoutQueue2(String message){
        System.err.println("消费者2：" + message + ", " + LocalTime.now());
    }
}
```
#### 给交换机发送消息
```Java
@Test
    public void testFanoutQueue(){
        String exchangeName = "hmall.fanout";
        String message = "hello,fanout";
        rabbitTemplate.convertAndSend(exchangeName,"",message);
    }
```
## Direct 交换机 - 定向路由
- Direct Exchange 会将接收到的消息根据规则路由到指定的Queue，因此称为定向路由
  - 每一个Queue都与Exchange设置一个BindingKey
  - 发布者发送消息时，指定消息的RoutingKey
  - Exchange将消息路由到BindingKey与消息RoutingKey一致的队列
### 使用
#### 创建队列
#### 创建交换机
![direct1.png](direct1.png)
![direct2.png](direct2.png)
#### 接收消息
```Java
 @RabbitListener(queues = "direct.queue1")
    public void listenerDirectQueue1(String message){
        System.err.println("消费者1：" + message + ", " + LocalTime.now());
    }
    @RabbitListener(queues = "direct.queue2")
    public void listenerDirectQueue2(String message){
        System.err.println("消费者2：" + message + ", " + LocalTime.now());
    }
```
#### 发送消息
```Java
@Test
    public void testDirectQueue(){
        String exchangeName = "hmall.direct";
        String message = "hello,direct";
        rabbitTemplate.convertAndSend(exchangeName,"red",message);
    }
```
## Topic交换机
- TopicExchange也是基于RoutingKey做消息路由，但是RoutingKey通常是多个单词组合，并且以.分割
- Queue与Exchange指定BindingKey时可以使用通配符
  - '#' 代指0个或多个单词
  - '*' 代指一个单词
## 声明队列交换机
- SpringAMQP提供了几个类，用来声明队列，交换机及其绑定关系
  - Queue：用于声明队列，可以用工厂类QueueBuilder构建
  - Exchange：用于声明交换机，可以用工厂类ExchangeBuilder构建
  - Binding：用于声明队列和交换机的绑定关系，可以用工厂类BindingBuilder构建
![mq3.png](mq3.png)
### 代码
```Java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author yuhao
 * @time 2024/6/9 23:25
 **/
@Configuration
public class FanoutConfiguration {
    @Bean
    public FanoutExchange fanoutExchange(){
        return ExchangeBuilder.fanoutExchange("hmall2.fanout").build();
    }
    @Bean
    public Queue fanoutQueue(){
        return QueueBuilder.durable("fanout.queue1").build();
    }
    @Bean
    public Binding fanoutQueue1Binding(Queue fanoutQueue,FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue).to(fanoutExchange);
    }
}
```
### 注解
```Java
@RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "direct.queue3"),
            exchange = @Exchange(name = "hmall.direct",type = ExchangeTypes.DIRECT),
            key = {"red","blue"}
    ))
    public void listenerDirectQueue3(String message){
        System.err.println("消费者3：" + message + ", " + LocalTime.now());
    }
```
## 消息转换器
- Spring的对消息对象的处理是由org.springframework.amqp.support.converter.MessageConverter来处理的
- 而默认实现是SimpleMessageConverter，基于JDK的ObjectOutputStream完成序列化的
- 存在以下问题
  - JDK的序列化有安全风险
  - JDK序列化的消息太大
  - JDK序列化的消息可读性差
- 建议采用JSON序列化代替默认的JDK序列化
  - 在publisher和consumer中都要引入jackson依赖
```xml
<dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
```
  - 在publisher和consumer中都要配置MessageConverter
```Java
 @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
```