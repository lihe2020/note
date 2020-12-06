#### 1. IO模型

##### 1.1 同步IO

示例代码：

```java
ExecutorService threadPool = Executors.newCachedThreadPool();
// 初始化socket连接
ServerSocket server = new ServerSocket();
server.bind(new InetSocketAddress("localhost", 6666));
while (true) {
    // 阻塞接收请求
    Socket socket = server.accept();
    // 将请求提交到线程池进行处理
    threadPool.submit(() -> {
        // 读取输入流
        try (InputStream input = socket.getInputStream()) {
            String result = StreamUtils.copyToString(input,
                    Charset.defaultCharset());
            
            System.out.println(result);
            
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    });
}
```

##### 1.2 NIO

NIO全称Non-Blocking IO，同步非阻塞IO。NIO有三大核心组件：Channel（通道）、Buffer（缓冲区）、Selector（选择器）。

###### 1.2.1 Buffer使用示例

```java
// 创建int缓冲区
IntBuffer ib = IntBuffer.allocate(5);
for (int i = 0; i < 5; i++) {
    // 向缓冲区写入数据
    ib.put(i + 1);
}
// 重置缓冲区标识位，类似于将模式从写切换到读
ib.flip();
while (ib.remaining() > 0) {
    System.out.println(ib.get());
}
```

重要说明：

- `MappedByteBuffer` 堆外缓冲区，它的内容是文件到虚拟内存的映射，用于修改文件对应的堆外内存，而无需将文件内容拷贝到java进程。

###### 1.2.2 Channel使用示例

```java
// 创建文件通道
FileOutputStream fs = new FileOutputStream("./out.txt");
FileChannel channel = fs.getChannel();
// 初始化缓冲区
ByteBuffer buffer = ByteBuffer.allocate(1024);
// 写入数据
buffer.put("hello world。".getBytes());
// 将缓冲区切换到读模式
buffer.flip();
// 从缓冲区读取字节流并写入通道
channel.write(buffer);
```

###### 1.2.3 Selector使用示例

Selector能够检测到多个通道是否有事件发生。

NIO服务端示例：

```java
// 创建selector
val selector = Selector.open();

val ssc = ServerSocketChannel.open();
// 绑定端口
ssc.bind(new InetSocketAddress("localhost", 6666));
// 设置成非阻塞模式
ssc.configureBlocking(false);
// 注册到selector，关注accept事件
ssc.register(selector, SelectionKey.OP_ACCEPT);

val timeout = Duration.ofSeconds(1).toMillis();

// 循环等待客户端连接
while (true) {

    // 等待1s
    if (selector.select(timeout) == 0) {
        continue;
    }

    // 读取所有selected key（我的理解是产生事件的通道对应的keys）
    val iterator = selector.selectedKeys()
            .iterator();
  
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();

        // 检测是否有新连接
        if (key.isAcceptable()) {

            // 获取一个连接通道
            SocketChannel socketCh = ssc.accept();
            socketCh.configureBlocking(false);
            // 将获取的通道注册到selector
            // 并关注read事件
            socketCh.register(selector, SelectionKey.OP_READ,
                    ByteBuffer.allocate(1024));
        }

        // 检测是否有read事件
        if (key.isReadable()) {
            val socketCh = (SocketChannel) key.channel();
            val buffer = (ByteBuffer) key.attachment();
            
            // 从通道中读取字节流
            socketCh.read(buffer);

            // 输出到控制台
            System.out.println(new String(buffer.array()));
        }

        // 处理完成后，移除当前key
        iterator.remove();
    }
}
```

##### 1.3 AIO模型

xxx

#### 2. 零拷贝

传统的一次IO读写操作会经过以下步骤：

- 上下文由用户态切换到内核态
- 通过DMA copy将文件拷贝到内核缓冲区（DMA全称direct memory access，直接内存拷贝，不经过CPU）
- 通过CPU将内核缓冲区拷贝到用户态缓冲区，上下文由内核态切到用户态
- 在用户态修改完成后，再通过CPU将用户态缓冲区拷贝到内核态，上下文由用户态切到内核态
- 最后通过DMA将内核态缓冲区刷新到IO中

从上面步骤可以看出，传统IO会有4次拷贝，3次上下文切换。

为了解决上述问题，Linux提供了mmap技术，它将文件映射到内核缓冲区，用户空间可以共享内核空间的数据，避免了内核态与用户态的拷贝，但修改时仍然会进行上下文切换，而且从内核态读取到从内核态刷新到IO仍然需要经过一次CPU拷贝，所以会有3次拷贝，3次上下文切换。而我们所说的**零拷贝指的是CPU拷贝次数是零**，所以mmap并不是真正意义上的零拷贝。

在Linux中还提供了sendFile函数，此时数据完全不经过用户态，直接从内核缓冲区进入Socket Buffer，因此就减小了一次上下文切换，但仍然需要3次拷贝，2次上下文切换。

为此，在Linux2.4中，对sendFile进行了优化，避免了CPU拷贝，从而基本上实现了零拷贝。在Java中，使用`transferTo`来进行零拷贝：

```java
try (SocketChannel clientSocketCh = SocketChannel.open()) {
  clientSocketCh.connect(new InetSocketAddress("localhost", 6666));

  FileInputStream inputStream = new FileInputStream("test.zip");
  FileChannel channel = inputStream.getChannel();
  // 将文件通道通过零拷贝传输到socket通道
  channel.transferTo(0, channel.size(), clientSocketCh);
}
```

#### 3. Netty概述

JDK自带的NIO技术存在问题：

- API繁杂，使用麻烦，学习、使用门槛较高
- 需要掌握Java多线程技术，熟悉网络编程才能编写高质量的NIO程序
- 需要精通网络访问的解决方案：如客户端断开重连、网络闪断、半包读写、网络拥塞和异常流量处理等等
- NIO本身的Epoll Bug，它会导致Selector空轮询，使CPU飚升，该问题直到JDK 1.7仍然存在

##### 3.1 Reactor编程模式

- 单Reator单线程
- 单Reator多线程
- 主从Reator多线程

##### 3.2 Netty模型

服务端示例：

```java
NioEventLoopGroup bossGroup = new NioEventLoopGroup();
NioEventLoopGroup workerGroup = new NioEventLoopGroup();

try {
    ServerBootstrap bootstrap = new ServerBootstrap();
    bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 128)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    // 添加处理程序
                    ch.pipeline().addLast(new MyHandler());
                }
            });

    // 绑定端口并启动服务器，同步等待服务器初始化完成
    ChannelFuture future = bootstrap.bind("localhost", 6666).sync();
    // 同步等待服务器关闭
    future.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}

class MyHandler extends ChannelInboundHandlerAdapter {
        // 读取客户端发送来的数据
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            ByteBuf buf = (ByteBuf) msg;
            System.out.printf("data receive: %s\n", buf.toString());
        }

        // 数据读取完毕
        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) {
            ctx.writeAndFlush(
                    Unpooled.copiedBuffer("send data to client", CharsetUtil.UTF_8)
            );
        }

        // 异常处理
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            ctx.close();
        }
    }
```

客户端示例：

```java
NioEventLoopGroup eventLoopGroup = new NioEventLoopGroup();
try {
    Bootstrap bootstrap = new Bootstrap();
    bootstrap.group(eventLoopGroup)
            .channel(NioSocketChannel.class)
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {

                        // 通道准备就绪
                        @Override
                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                            ctx.writeAndFlush(Unpooled.copiedBuffer("hello, server.",
                                    CharsetUtil.UTF_8));
                        }

                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            ByteBuf buf = (ByteBuf) msg;
                            System.out.printf("data receive from server: %s\n", 
                                    buf.toString(CharsetUtil.UTF_8));
                        }
                    });
                }
            });

    ChannelFuture future = bootstrap.connect("localhost", 6666).sync();
    future.channel().closeFuture().sync();
} finally {
    eventLoopGroup.shutdownGracefully();
}
```

##### 3.3 TaskQueue

##### 3.4 Netty心跳机制