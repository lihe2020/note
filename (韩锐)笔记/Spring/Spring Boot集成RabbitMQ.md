
## ConnectionFactory

1. CachingConnectionFactory
   1. 需要注意ChannelSize
   2. 适当时候需要自定义executor，因为底层单独为每个连接创建5个线程
   3. 如果生产/消费者用的同一个连接，那么有可能会产生死锁，解决办法就是为生产、消费者分别创建连接
   4. 为确保消息发送成功，需要配置`publisherConfirms`和`publisherReturns`，参见[Introducing Publisher Confirms](https://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/)。

2. SingleConnectionFactory
  仅用于单元测试

3. SimpleRoutingConnectionFactory
  为每个ConnectionFactory定义一个键，方便在运行时动态查找相关连接工厂，适用于多个ConnectionFactory场景。

4. LocalizedQueueConnectionFactory
  在配置了高可用（HA）情况下，它总是选择连接到主节点。
  目前我们（线上/线下）没配置高可用/负载均衡

## AmqpTemplate

AmqpTemplate提供了对发送/接收消息的抽象

1. RabbitTemplate
2. RabbitMessagingTemplate
  提供了与Spring Framework消息抽象的集成，适用于与Spring其他项目集成。
3. BatchingRabbitTemplate
  批量发送消息

## 发送消息

1. 如果没有定义exchange或routingkey，那么AmqpTemplate将默认其为空字符串
2. 如果只指定routingkey，那么消息将被发送到与routingkey同名的队列。因为AMQP规范规定了一个没有名字的默认exchange，类型是Direct，并且所有队列自动绑定到该exchange，routingkey就是队列名字。
3. 可以使用`MessageBuilder`和`MessagePropertiesBuilder`创建更加详细 的消息和消息属性。

## 接收消息

### Container

1. SimpleMessageListenerContainer/SimpleRabbitListenerContainerFactory
2. DirectMessageListenerContainer/DirectRabbitListenerContainerFactory

### 消息转换

消息转换（`MessageConverter`）专用于方法级别的`@RabbitListener`，用于在处理消息之前将原始消息转换成相应的java类型

1. SimpleMessageConverter 将消息转换成`String`和`java.io.Serializable`，其他情况保持`byte[]`不变
2. GenericMessageConverter 继承自SimpleMessageConverter，用于需要自定义转换服务（`ConversionService`）的场景
3. Jackson2JsonMessageConverter
4. ContentTypeDelegatingMessageConverter

### 方法签名

应用了`@RabbitListener`注解的方法除了接收消息对象外，还支持注入关于消息的一些其他数据，包括：

1. `org.springframework.amqp.core.Message`对象
2. `com.rabbitmq.client.Channel`对象
3. `@Header`，使用@Header注解来提取特定的Header数据，如队列名（`AmqpHeaders.CONSUMER_QUEUE`）等
4. `@Headers`，使用@Headers注解来获取所有的Header数据，注意方法的签名必须实现了`java.util.Map`接口
5. 没有使用任何注解的参数，且不是Message或Channel类型，那么将被认为是消息本身，您也可以使用`@Payload`来明确标记

### 方法返回值

如果您的监听方法有返回值，那么

### 异常处理

默认情况下，如果监听方法抛出了异常，那么消息将会重新进入队列和投递，或者路由到死信队列。

