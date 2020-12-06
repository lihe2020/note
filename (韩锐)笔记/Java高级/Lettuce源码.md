梳理：

- 初始化Bootstrap：ConnectionBuilder

  - 由于RedisClusterClient的createConnectionBuilder方法触发

- 初始化group：KQueueEventLoopGroup

  - 由于RedisClusterClient的createConnectionBuilder方法触发

- 初始化Channel：KQueueSocketChannel

- 初始化Handler：PlainChannelInitializer

  - 包括

    1. ChannelGroupListener
    2. CommandEncoder
    3. CommandHandler（在RedisClusterClient的connectToNodeAsync方法中注册）
    4. ConnectionWatchdog
    5. ConnectionEventTrigger
    6. 

  - 由AbstractRedisClient的initializeChannelAsync0方法触发

    ```java
    public abstract class AbstractRedisClient {
      ...
      
      private void initializeChannelAsync0(ConnectionBuilder connectionBuilder, CompletableFuture<Channel> channelReadyFuture,
                SocketAddress redisAddress) {
        
        logger.debug("Connecting to Redis at {}", redisAddress);
    
        Bootstrap redisBootstrap = connectionBuilder.bootstrap();
    
        RedisChannelInitializer initializer = connectionBuilder.build();
        // 初始化handler
        redisBootstrap.handler(initializer);
        
        clientResources.nettyCustomizer().afterBootstrapInitialized(redisBootstrap);
        CompletableFuture<Boolean> initFuture = initializer.channelInitialized();
        // 连接
        ChannelFuture connectFuture = redisBootstrap.connect(redisAddress);
        
        ...
      }
    }
    
    public class ConnectionBuilder {
      ...
      
      // 创建Netty handler
      public RedisChannelInitializer build() {
            return new PlainChannelInitializer(pingCommandSupplier, this::buildHandlers, clientResources, timeout);
        }
      
      protected List<ChannelHandler> buildHandlers() {
    				...
            List<ChannelHandler> handlers = new ArrayList<>();
    
            connection.setOptions(clientOptions);
    
            handlers.add(new ChannelGroupListener(channelGroup));
            handlers.add(new CommandEncoder());
            handlers.add(commandHandlerSupplier.get());
    
            if (clientOptions.isAutoReconnect()) {
                handlers.add(createConnectionWatchdog());
            }
    
            handlers.add(new ConnectionEventTrigger(connectionEvents, connection, clientResources.eventBus()));
    
            if (clientOptions.isAutoReconnect()) {
                handlers.add(createConnectionWatchdog());
            }
    
            return handlers;
        }
    }
    ```

- 连接

  - 由AbstractRedisClient的initializeChannelAsync0方法触发

