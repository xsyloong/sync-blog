> B站课程：
>
> https://www.bilibili.com/video/BV1py4y1E7oA
>
> 《**黑马程序员：netty深入浅出Java网络编程教程**》
>
> 课程资料： [[Netty01-nio]] [[Netty02-netty入门]] [[Netty03-netty进阶]] [[Netty04-优化与源码]]

## **Netty**

### **NIO 基本组件**

##### **ServerSocketChannel**

做服务器的数据传输通道

##### **SocketChannel**

服务器和客户端都可作为数据传输通道

#### **NIO非阻塞模式**

##### **configureBlocking()**

- 在ServerSocketChannel实例中，`serverSocketChannel.configureBlocking(true)`即可配置非阻塞模式，则`accept()`获取新链接**（SocketChannel实例）**的之前，不会在阻塞，非阻塞没有链接建立，***accept（）\***返回null。
- 在**SocketChannel**实例中设置非阻塞模式，则读数据时也不会阻塞。没读到数据时返回0；

##### **Selector**

> 单纯的非阻塞模式，只是没有接收到数据或链接的时候不阻塞了，但会无限空转，消耗资源，因此有了Selector,对channel的行为进行管理。无事件时阻塞，有事件时运行。

- 处理**accpet**

  ```java
          try (Selector selector = Selector.open();
               ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
              serverSocketChannel.configureBlocking(true);
  
              // 将serverSocketChannel注册到selector中，获取与serverSocketChannel相关的key，设置key感兴趣的事件。
              // 通过该key可以知道是哪个channel发生的什么事件
              SelectionKey key = serverSocketChannel.register(selector, 0， null);
              // 四种事件类型，1. accept - 有链接请求 2. connect - 建立连接 3. read - 可读事件  4. write - 可写事件
              key.interestOps(SelectionKey.OP_ACCEPT);
  
              serverSocketChannel.bind(new InetSocketAddress(9999));
              while(true){
                  // selector获取发生的事件，有事件执行，没事件阻塞
                  selector.select();
                  // 遍历所有获取的事件
                  Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                  while(iterator.hasNext()){
                      // 如果有accept事件发生 ， 该key就是上述serverSocketChannel的key，channel就是serverSocketChannel
                      SelectionKey next = iterator.next();
                      ServerSocketChannel serverSocketChannel1 = (ServerSocketChannel) next.channel();
                      // 处理链接事件,建立链接，获取与链接相关的客户端的channel，
                      // 如果事件没有被处理，则selector会一直把他当作新事件等待处理，无限循环， 
                      // 使用key.cancel();  可以取消该事件，以免不处理被selector识别为新事件
                      SocketChannel sc = channelled.accept();
                  }
              }
          } catch (Exception e) {
              e.printStackTrace();
          }
  ```

  ![[image-20250603093725572.png]]

- 处理**读事件**

  ```java
             while(true){
                  // selector获取发生的事件，有事件执行，没事件阻塞
                  selector.select();
                  // 遍历所有获取的事件
                  Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                  while(iterator.hasNext()){
                      SelectionKey nextKey = iterator.next();
                      iterator.remove();// 删除key在selectedKeys集合中，因为selectedKeys不会主动删除key,以后每次循环都会带着该key
                      // 区分事件类型
                      if(nextKey.isAcceptable()){
                          ServerSocketChannel ssc = (ServerSocketChannel) nextKey.channel();
                          SocketChannel accept = ssc.accept();
                          SelectionKey acceptKey = accept.register(selector, 0, null);
                          accept.configureBlocking(false);
                          acceptKey.interestOps(SelectionKey.OP_READ);
                     } else if (nextKey.isReadable()) {
                          try {
                              ByteBuffer buffer = ByteBuffer.allocate(16);// 分配空间的大小会决定触发读事件的次数，直到全部读结束
                              SocketChannel sc = (SocketChannel) nextKey.channel();
                              int read = sc.read(buffer);
                              if(read == -1) key.cancel(); // 当客户端正常断开链接时，就根据是否读到-1来判断是否断开
                              else{
                                  buffer.flip();// 切换到读模式
                             }
                         } catch (Exception e){   // 当客户端异常断开时，根据异常来判断，因为不管正常断开还是异常断开，都会产生新的读事件，导致机器一直循环
                              e.printStackTrace();
                              nextKey.cancel();
                         }
                     }
  
                 }
             }
  ```

- 处理**消息的边界 - 这是个比较重要的问题**
  - 就是处理**ByteBuffer**的大小的问题，太大或太小会有粘包半包等问题。
    - 第一种思路：服务端与客户端协商一种固定大小，实际上也不会解决上述问题
    - 第二种思路：客户端在一条消息后使用一个特殊的分隔符分割，服务器端检测到分隔符就用新的ByteBuffer来接收剩余的消息。
    - ***第三种思路\***：客户端消息分为两个部分，第一部分使用一个固定长度来保存剩余消息的总长度，第二部分就是实际的消息。然后服务端根据第一部分来生成指定大小的ByteBuffer来接受剩余的消息。

- 处理**消息边界** - **容量超出**
  - 在处理读事件的代码中，ByteBuffer作为接收消息的缓存确是**局部变量**，如果消息太长，会导致多次读取，而且多次读取过程中如果缓存满了则会导致后续读取的消息覆盖缓存中现有的内容，一眼能看出的**解决方案**是，缓存作为读取事件的**全局变量**来使用，实际上这种方式也是不合理的，接下来就有了实际的解决方案。

- 处理**消息边界 - 附件 - attachment**

  - ByteBuffer buffer = ByteBuffer.allocate(16);
    SelectionKey acceptKey = accept.register(selector, 0, buffer);
  - `register(Selector，ops, attachment)`函数的第三个参数就是给channel注册一个缓冲区
  - 发生读事件时，从key里面拿出buffer进行操作`ByteBuffer buffer = (ByteBuffer) nextKey.attachment();`
  - **扩容：**
  - 创建新的ByteBuffer来都代替旧的ByteBuffer,注意将旧的内容拷贝进来

  ```java
                    } else if (nextKey.isReadable()) {
                          try {
                              SocketChannel sc = (SocketChannel) nextKey.channel();
                              ByteBuffer buffer = (ByteBuffer) nextKey.attachment();
                              int read = sc.read(buffer);
                              if(read  -1) nextKey.cancel(); // 当客户端正常断开链接时，就根据是否读到-1来判断是否断开
                              else{
                                  buffer.flip();// 切换到读模式
                                  // 进行读操作
                                  // 读取后查看ByteBuffer
                                  buffer.compact(); // 合并前面已经读取掉的内容
                                  if(buffer.position()  buffer.limit()){ // 读取后发现ByteBuffer还是满的 说明需要扩容
                                      ByteBuffer newBuffer = ByteBuffer.allocate(buffer.limit() * 2);
                                      newBuffer.put(buffer);
                                      nextKey.attach(newBuffer);
                                 }
                             }
                         }
  ```

- **ByteBuffer大小分配**

  - 每个channel必须维护一个独立的ByteBuffer
  - 容量不能太大，随着链接过多会导致占据大量内存，**一种思路**：一般都设置一个可变的ByteBuffer，先设置小的，容量不够再设置大的，把小的拷贝进去。**另外一种思路（netty实现）**：使用数组组成buffer，当一个数组不够时使用心得数组来存剩余的从而避免的拷贝。

- 处理**可写事件**

  - 为什么有可写事件，当服务器需要给客户端写入的数据过多的时候，数据不可能一次性写完，当网络的发送能力达到限制，并且网络发送缓冲区塞满的时候，此时就不会在发送。如果此时你设计的是轮询发送直到发送完毕，会因为等待网络发送导致浪费大量资源，因此延伸出处理可写事件。

    ```java
    // 服务器端
        public static void main(String[] args) {
            try(ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
                Selector selector = Selector.open();
           ) {
                serverSocketChannel.configureBlocking(false);
                SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
                serverSocketChannel.bind(new InetSocketAddress(9999));
                while(true){
                    selector.select();
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while(iterator.hasNext()){
                        SelectionKey nextKey = iterator.next();
                        iterator.remove();
                        if(nextKey.isAcceptable()){
                            ServerSocketChannel ssc = (ServerSocketChannel) nextKey.channel();
                            SocketChannel sc = ssc.accept();
                            sc.configureBlocking(false);
                            SelectionKey scKey = sc.register(selector, SelectionKey.OP_READ);
    
                            StringBuilder builder = new StringBuilder();
                            for(int i = 0;i < 3000000;i++){
                                builder.append('a');
                           }
                            ByteBuffer encode = StandardCharsets.UTF_8.encode(builder.toString()); // 1. 需要被发送的大数据
    
                            int count = sc.write(encode); //2. 先写入一些数据
                            System.out.println(count);
    
                            if(encode.hasRemaining()) //3. 如果buffer还有剩余，没有被发送完毕
                           {
                                scKey.interestOps(scKey.interestOps() | SelectionKey.OP_WRITE);//4. 设置感兴趣写事件,使用位运算来代表即感兴趣读事件又感兴趣写事件
                                scKey.attach(encode);
                           }
                       } else if (nextKey.isWritable()){
                            SocketChannel sc = (SocketChannel) nextKey.channel();
                            int write = sc.write((ByteBuffer) nextKey.attachment());
                            System.out.println(write);
                            if(!((ByteBuffer) nextKey.attachment()).hasRemaining()){
                                nextKey.attach(null);
                                nextKey.interestOps(nextKey.interestOps() - SelectionKey.OP_WRITE);
                           }
                       }
                   }
               }
    
           } catch (Exception e) {
                e.printStackTrace();
           }
    
       }
    // 客户端
        public static void main(String[] args) {
            try (SocketChannel sc = SocketChannel.open()){
                sc.connect(new InetSocketAddress(9999));
                int count = 0;
                while(true){
                    ByteBuffer allocate = ByteBuffer.allocate(160);
                    count = sc.read(allocate);
                    System.out.println(count);
                    allocate.clear();
               }
           } catch (IOException e) {
                throw new RuntimeException(e);
           }
       }
    ```

#### NIO 多线程优化

##### **使用多线程优化**

- 单线程配一个选择器，专门处理accept事件

- 创建Cpu核心数多线程，每个线程创建一个selector，专门读写事件

- 模型![[image-20250603093745669.png]]

- 实践

  - 服务端

    ```java
    package org.learn.nio.server;
    
    import java.io.IOException;
    import java.net.InetSocketAddress;
    import java.nio.ByteBuffer;
    import java.nio.channels.*;
    import java.nio.charset.StandardCharsets;
    import java.util.Iterator;
    import java.util.concurrent.ConcurrentLinkedQueue;
    
    public class MultiThreadServer {
        public static void main(String[] args) {
            try (ServerSocketChannel ssc = ServerSocketChannel.open();
                 Selector boss = Selector.open()) {
                ssc.bind(new InetSocketAddress(9999));
                ssc.configureBlocking(false);
                ssc.register(boss, SelectionKey.OP_ACCEPT);
                Worker worker = new Worker("worker-0");
    
                while (true) {
                    boss.select();
                    Iterator<SelectionKey> iterator = boss.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        SelectionKey key = iterator.next();
                        iterator.remove();
                        if (key.isAcceptable()) {
                            ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
                            SocketChannel sc = serverChannel.accept();
                            sc.configureBlocking(false); // 必须设置为非阻塞
                            worker.register(sc);
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    
        static class Worker implements Runnable {
            private Thread thread;
            private Selector selector;
            private volatile boolean started = false;
            private String name;
            private ConcurrentLinkedQueue<Runnable> queue = new ConcurrentLinkedQueue<>();
    
            Worker(String name) {
                this.name = name;
            }
    
            public void register(SocketChannel sc) throws IOException {
                if (!started) {
                    selector = Selector.open();
                    thread = new Thread(this, name);
                    thread.start();
                    started = true; // 标记线程已启动
                }
                /**
                 * 由于worker和boss线程是并行执行，所以会导致boss线程调用注册方法后，由于此时worker已经阻塞了，所以会丢弃第一次客户端的发送的事件，客户端在发送后才会唤醒worker
                 */
                queue.add(() -> {
                    try {
                        sc.register(selector, SelectionKey.OP_READ);
                        System.out.println("客户端注册成功: " + sc.getRemoteAddress());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
                selector.wakeup(); // 唤醒 select()
            }
    
            @Override
            public void run() {
                while (true) {
                    try {
                        selector.select();
                        // 处理队列中的任务（如注册 OP_READ）
                        Runnable task;
                        while ((task = queue.poll()) != null) {
                            task.run();
                        }
                        // 处理已就绪的 Channel
                        Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                        while (iterator.hasNext()) {
                            SelectionKey key = iterator.next();
                            iterator.remove();
                            if (key.isReadable()) {
                                SocketChannel channel = (SocketChannel) key.channel();
                                ByteBuffer buffer = ByteBuffer.allocate(16);
                                int read = channel.read(buffer);
                                if (read == -1) {
                                    key.cancel();
                                    channel.close();
                                    System.out.println("客户端断开连接");
                                } else {
                                    buffer.flip();
                                    System.out.println("收到数据: " +
                                                       StandardCharsets.UTF_8.decode(buffer));
                                }
                            }
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
    ```

  - 客户端

    ```java
    package org.learn.nio.client;
    
    import java.io.IOException;
    import java.net.InetSocketAddress;
    import java.nio.ByteBuffer;
    import java.nio.channels.SocketChannel;
    import java.nio.charset.StandardCharsets;
    
    public class WriteClient {
        public static void main(String[] args) throws IOException {
            try (SocketChannel channel = SocketChannel.open()) {
                channel.connect(new InetSocketAddress("127.0.0.1",9999));
                ByteBuffer encode = StandardCharsets.UTF_8.encode("hello world");
                channel.write(encode);
            } catch (Exception e){
                e.printStackTrace();
            }
        }
    }
    ```



------

### Netty

#### netty实例

- netty依赖

  ```xml
          <dependency>
              <groupId>io.netty</groupId>
              <artifactId>netty-all</artifactId>
              <version>4.1.66.Final</version>
          </dependency>
  ```

- 服务端

  ```java
  package org.learn.netty;
  
  import io.netty.bootstrap.ServerBootstrap;
  import io.netty.channel.ChannelHandlerContext;
  import io.netty.channel.ChannelInboundHandlerAdapter;
  import io.netty.channel.ChannelInitializer;
  import io.netty.channel.nio.NioEventLoopGroup;
  import io.netty.channel.socket.nio.NioServerSocketChannel;
  import io.netty.channel.socket.nio.NioSocketChannel;
  import io.netty.handler.codec.string.StringDecoder;
  import org.apache.logging.log4j.LogManager;
  import org.apache.logging.log4j.Logger;
  
  public class NttyServer {
      private static final Logger log = LogManager.getLogger(NttyServer.class);
  
      public static void main(String[] args) {
          new ServerBootstrap()
                  .group(new NioEventLoopGroup())
  // 创建 NioEventLoopGroup，可以简单理解为 `线程池 + Selector` 后面会详细展开
                  .channel(NioServerSocketChannel.class)
  // 选择服务 Scoket 实现类，其中 NioServerSocketChannel 表示基于 NIO 的服务器端实现
                  .childHandler(new ChannelInitializer<NioSocketChannel>() {
  //为啥方法叫 childHandler，是接下来添加的处理器都是给 SocketChannel 用的，而不是给 ServerSocketChannel。ChannelInitializer 处理器（仅执行一次），它的作用是待客户端 SocketChannel 建立连接后，执行 initChannel 以便添加更多的处理器
                      @Override
                      protected void initChannel(NioSocketChannel ch) {
                          ch.pipeline().addLast(new StringDecoder());
  // SocketChannel 的处理器，解码 ByteBuf => String
                          ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
  // SocketChannel 的业务处理器，使用上一个处理器的处理结果
                              @Override
                              public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                  log.info(msg);
                              }
                          });
                      }
                  }).bind(8080);
  // ServerSocketChannel 绑定的监听端口
      }
  }
  ```

- 客户端

  ```java
  package org.learn.netty;
  
  import io.netty.bootstrap.Bootstrap;
  import io.netty.channel.ChannelInitializer;
  import io.netty.channel.nio.NioEventLoopGroup;
  import io.netty.channel.socket.nio.NioSocketChannel;
  import io.netty.handler.codec.string.StringEncoder;
  
  import java.net.InetSocketAddress;
  
  
  public class NettyClient {
      public static void main(String[] args) throws InterruptedException {
          new Bootstrap()
                  .group(new NioEventLoopGroup())
  // 创建 NioEventLoopGroup，同 Server
                  .channel(NioSocketChannel.class)
  // 选择客户 Socket 实现类，NioSocketChannel 表示基于 NIO 的客户端实现
                  .handler(new ChannelInitializer<NioSocketChannel>() {
  // 添加 SocketChannel 的处理器，ChannelInitializer 处理器（仅执行一次），它的作用是待客户端 SocketChannel 建立连接后，执行 initChannel 以便添加更多的处理器
                      @Override
                      protected void initChannel(NioSocketChannel ch) throws Exception {
                          ch.pipeline().addLast(new StringEncoder());
  //消息会经过通道 handler 处理，这里是将 String => ByteBuf 发出，数据经过网络传输，到达服务器端，服务器端 handler 先后被触发，走完一个流程
                      }
                  })
                  .connect(new InetSocketAddress("localhost",8080))
  // 指定要连接的服务器和端口
                  .sync()
  // Netty 中很多方法都是异步的，如 connect，这时需要使用 sync 方法等待 connect 建立连接完毕
                  .channel()
  // 获取 channel 对象，它即为通道抽象，可以进行数据读写操作
                  .writeAndFlush("hello world");
  // 写入消息并清空缓冲区
      }
  }
  ```

- Netty 应用流程![[image-20250603093838489.png]]

  > **💡\*注意\***
  >
  > - 把 channel 理解为数据的通道
  > - 把 msg 理解为流动的数据，最开始输入是 ByteBuf，但经过 pipeline 的加工，会变成其它类型对象，最后输出又变成 ByteBuf
  > - 把 handler 理解为数据的处理工序
  >   - 工序有多道，合在一起就是 pipeline，pipeline 负责发布事件（读、读取完成...）传播给每个 handler， handler 对自己感兴趣的事件进行处理（重写了相应事件处理方法）
  >   - handler 分 Inbound 和 Outbound 两类
  > - 把 eventLoop 理解为处理数据的工人
  >   - 工人可以管理多个 channel 的 io 操作，并且一旦工人负责了某个 channel，就要负责到底（绑定）
  >   - 工人既可以执行 io 操作，也可以进行任务处理，每位工人有任务队列，队列里可以堆放多个 channel 的待处理任务，任务分为普通任务、定时任务
  >   - 工人按照 pipeline 顺序，依次按照 handler 的规划（代码）处理数据，可以为每道工序指定不同的工人

#### Netty 组件

##### `EventLoop`- 数据处理工

> **💡\*简介\***
>
> EventLoop 本质是一个单线程执行器（同时维护了一个 Selector），里面有 run 方法处理 Channel 上源源不断的 io 事件。
>
> 它的继承关系比较复杂
>
> 1. 一条线是继承自 j.u.c.ScheduledExecutorService 因此包含了线程池中所有的方法
> 2. 另一条线是继承自 netty 自己的 OrderedEventExecutor（有序的事件执行器）
>    - 提供了 boolean inEventLoop(Thread thread) 方法判断一个线程是否属于此 EventLoop
>    - 提供了 parent 方法来看看自己属于哪个 EventLoopGroup
>
> 
>
> **EventLoopGroup** 是一组 EventLoop，Channel 一般会调用 EventLoopGroup 的 register 方法来***绑定\***其中一个 EventLoop，后续这个 Channel 上的 io 事件都由此 EventLoop 来处理（保证了 io 事件处理时的线程安全）（IO操作都是同一个线程来处理）
>
> 3. 继承自 netty 自己的 EventExecutorGroup
>    - 实现了 Iterable 接口提供遍历 EventLoop 的能力
>    - 另有 next 方法获取集合中下一个 EventLoop
>  

- `EventLoop`  执行任务

  ```java
          NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
          // 1. io任务，普通任务， 定时任务
          // 2. 参数可传入线程数，线程数也代表该组有多少 EventLoop实例
          // 方法
          //    1. 获取下一个 EventLoop 对象
          nioEventLoopGroup.next();
          //    2. 对EventLoop 对象提交普通任务，异步处理该普通任务  例如：accept事件后可以提交异步任务做处理
          nioEventLoopGroup.next().submit(() -> {
              System.out.println("ok");
          });
          //   3. 提交定时任务  例如：定时发送心跳，保活链接
          //   4. io任务就不演示，在netty的handler中就是拿出event loop进行处理IO任务
          nioEventLoopGroup.next().scheduleAtFixedRate(() -> {
              System.out.println("ok");
          }, 0, 10, TimeUnit.SECONDS);
  ```

- `EventLoop `分工细化

  - 简单细分

    ```java
            new ServerBootstrap()
                    // 两个 group 组， 分别代表boss group 和 worker group, boss专门负责处理accept事件，worker使用两个线程负责处理其余事件
                    // 其中Boss group 不用指定线程数，因为boss只会绑定NioServerSocketChannel，在此服务器中该channel只有一个，因此只会有一个EventLoop绑定。
                    .group(new NioEventLoopGroup(),new NioEventLoopGroup(2))
                    .channel(NioServerSocketChannel.class)
                    .handler(new ChannelInitializer<NioSocketChannel>() {
                        @Override
                        protected void initChannel(NioSocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new StringDecoder());
                        }
                    })
                    .bind(9999);
    ```

  - 进一步细分

    ```java
            public static void main(String[] args) {
                // 如果某些handler执行时间过长，运行时会导致阻塞其他的该EventLoop管理的Channel组。因此考虑用其他的异步线程来处理
                // *** DefaultEventLoop 可处理普通任务和定时任务，不可处理IO任务，因此用其来处理handler中的普通任务 
                DefaultEventLoop defaultEventLoop = new DefaultEventLoop();
                new ServerBootstrap()
                        .group(new NioEventLoopGroup(), new NioEventLoopGroup(2))
                        .channel(NioServerSocketChannel.class)
                        .childHandler(new ChannelInitializer<NioSocketChannel>() {
                            @Override
                            protected void initChannel(NioSocketChannel ch) throws Exception {
                                ch.pipeline().addLast("handler1",new ChannelInboundHandlerAdapter() {
                                    @Override
                                    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                        log.debug(((ByteBuf) msg).toString(Charset.defaultCharset()));
                                        ctx.fireChannelRead(msg);
                                        // *** 此方法是将该msg交给下一个handler来处理
                                    }
                                }).addLast(defaultEventLoop,"handler2",new ChannelInboundHandlerAdapter() {
                                    // *** 注意三个参数，第一个参数指定异步任务运行组，第二个起名字，第三个匿名类重写里面的方法
                                    @Override
                                    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                        log.debug(((ByteBuf)msg).toString(Charset.defaultCharset()));
                                    }
                                });
                            }
                        })
                        .bind(9999);
            }
    ```

  - 原理分析：handler 之间如何切换线程的![[image-20250603093902871.png]]

    - 关键代码 `io.netty.channel.AbstractChannelHandlerContext#invokeChannelRead()`

      ```java
      static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
          final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
          // 下一个 handler 的事件循环是否与当前的事件循环是同一个线程
          EventExecutor executor = next.executor(); // 调用下一个handler的EventLoop
          
          // 是，直接调用
          if (executor.inEventLoop()) {  // 当前handler中的线程 ， 是否和event loop是同一个线程
              next.invokeChannelRead(m);
          } 
          // 不是，将要执行的代码作为任务提交给下一个事件循环处理（换人）
          else {
              executor.execute(new Runnable() {
                  @Override
                  public void run() {
                      next.invokeChannelRead(m);
                  }
              });
          }
      }
      ```

      - 如果两个 handler 绑定的是同一个线程，那么就直接调用
      - 否则，把要调用的代码封装为一个任务对象，由下一个 handler 的线程来调用

##### Channel

- 主要方法
  - close() 可以用来关闭 channel
  - closeFuture() 用来处理 channel 的关闭
    - sync 方法作用是同步等待 channel 关闭
    - 而 addListener 方法是异步等待 channel 关闭
  - pipeline() 方法添加处理器
  - write() 方法将数据写入，因此netty有缓冲机制，写入缓冲区后不会立即发送，相应的有 flush()方法可立即将缓冲区内容发出
  - writeAndFlush() 方法将数据写入并刷出

###### Channel Future

- `Future` 对象

  - 在java并发中，Future对象是做为获取异步处理结果的数据集对象。

    ```java
    ExecutorService executor = Executors.newFixedThreadPool(4); 
    // 定义任务:
    Callable<String> task = new Task();
    // 提交任务并获得Future:
    Future<String> future = executor.submit(task);
    // 从Future获取异步执行返回的结果:
    String result = future.get(); // 可能阻塞
    // get()：获取结果（可能会等待）
    // get(long timeout, TimeUnit unit)：获取结果，但只等待指定的时间；
    // cancel(boolean mayInterruptIfRunning)：取消当前任务； 
    // isDone()：判断任务是否已完成。
    ```

  - 关于java并发编程，基础可看 https://liaoxuefeng.com/books/java/threading/basic/index.html

- `Channel Future` - 获取Channel异步方法的结果

  - `connect(new InetSocketAddress("localhost",8080))` - 建立链接时的异步

    ```java
            public static void main(String[] args) throws InterruptedException {
                // 获取异步处理结果
                ChannelFuture connect = new Bootstrap()
                        .group(new NioEventLoopGroup())
                        .channel(NioSocketChannel.class)
                        .handler(new ChannelInitializer<NioSocketChannel>() {
                            @Override
                            protected void initChannel(NioSocketChannel ch) throws Exception {
                                ch.pipeline().addLast(new StringEncoder());
                            }
                        })
                        // 由于建立链接是异步非阻塞处理的 ， 因此需要获取异步处理的结果。
                        .connect(new InetSocketAddress(9999));
                                        // 主动阻塞等待异步处理的结果。不主动阻塞
                Channel channel = connect.sync()
                                         // 获取channel
                                         .channel();
            }
    ```

  - `addListener()`获取异步处理结果

    ```java
                // 使用addListener方法处理异步结果。
                channelFuture.addListener(new ChannelFutureListener() {
                    @Override
    // 添加listener后 在nio线程链接建立好之后，就会调用operationComplete方法。参数内的future就是调用addListener方法的future
                    public void operationComplete(ChannelFuture future) throws Exception {
                        Channel channel = future.channel();
                        log.debug("{}",channel);
                        channel.writeAndFlush("hello world");
                    }
                });
    ```

  - `close()`- 处理channel异步关闭

    ```java
               ........ 省略重复逻辑
                new Thread(() -> {
                    Scanner scanner =new Scanner(System.in);
                    while(true){
                        String line = scanner.nextLine();
                        if("q".equals(line)){
                            // 做channel的关闭。此方法实际上是异步关闭，如果需要阻塞获取异步关闭的结果需要close future
                            channel.close();
                        }
                        channel.writeAndFlush(line);
                    }
                }).start();
                ChannelFuture closeFuture = channel.closeFuture();
    
                // 1. 主线程阻塞等待channel关闭，然后进行处理
                closeFuture.sync();
                log.info("做一些channel关闭之后的处理");
    
                // 2. 在nio线程中链接关闭后执行operationComplete，以及一些关闭后的操作
                closeFuture.addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        log.info("做一些channel关闭之后的处理");
               // 由于nio event loop组在channel关闭后， 内部仍有工作线程在工作， 调用该方法关闭这些所有工作线程整个程序才算关闭
               // 在此方法中， 工作线程会停止接收新的任务，等待所有任务全部执行完毕后自动关闭。
                        nioEventLoopGroup.shutdownGracefully();
                    }
                });
    ```

###### Future & Promise

主要用于异步处理时，首先要说明 netty 中的 Future 与 jdk 中的 Future 同名，但是是两个接口，netty 的 Future 继承自 jdk 的 Future，而 Promise 又对 netty Future 进行了扩展

- jdk Future 只能同步等待任务结束（或成功、或失败）才能得到结果
- netty Future 可以同步等待任务结束得到结果，也可以异步方式得到结果，但都是要等任务结束
- netty Promise 不仅有 netty Future 的功能，而且脱离了任务独立存在，只作为两个线程间传递结果的容器

| **功能/名称** | **jdk Future**                 | **netty Future**                                             | **Promise**  |
| ------------- | ------------------------------ | ------------------------------------------------------------ | ------------ |
| cancel        | 取消任务                       | -                                                            | -            |
| isCanceled    | 任务是否取消                   | -                                                            | -            |
| isDone        | 任务是否完成，不能区分成功失败 | -                                                            | -            |
| get           | 获取任务结果，阻塞等待         | -                                                            | -            |
| getNow        | -                              | 获取任务结果，非阻塞，还未产生结果时返回 null                | -            |
| await         | -                              | 等待任务结束，如果任务失败，不会抛异常，而是通过 isSuccess 判断 | -            |
| sync          | -                              | 等待任务结束，如果任务失败，抛出异常                         | -            |
| isSuccess     | -                              | 判断任务是否成功                                             | -            |
| cause         | -                              | 获取失败信息，非阻塞，如果没有失败，返回null                 | -            |
| addLinstener  | -                              | 添加回调，异步接收结果                                       | -            |
| setSuccess    | -                              | -                                                            | 设置成功结果 |
| setFailure    | -                              | -                                                            | 设置失败结果 |

*三个接口案例：

- jdk - Future

  ``````java
          public static void main(String[] args) throws ExecutionException, InterruptedException {
              ExecutorService executorService = Executors.newFixedThreadPool(2);
              Future<Integer> future = executorService.submit(new Callable<Integer>() {
                  @Override
                  public Integer call() throws Exception {
                      Thread.sleep(1000);
                      log.info("执行计算");
                      return 100;
                  }
              });
              log.info("异步结果：{}",future.get());
          }
  ``````

- netty - Future

  ```java
          public static void main(String[] args) throws ExecutionException, InterruptedException {
              NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
              EventLoop next = nioEventLoopGroup.next();
              io.netty.util.concurrent.Future<Integer> future = next.submit(() -> {
                  log.info("执行运算...");
                  Thread.sleep(1000);
                  return 100;
              });
              // 此处为同步获取处理结果，因为在获取结果的时候主线程阻塞等待执行线程执行结束
              // 1. 同步获取处理结果，还有一种异步获取处理结果
              // log.info("异步结果：{}",future.get());
  
              // 2. 异步获取处理结果   -- 可以写成lambda 暂不写，因为要看清是哪个接口
              // 因为addListener的listener是在操作执行完成之后执行的，因此此时数据结果已准备完毕，所以getNow()不会获取null
              future.addListener(new GenericFutureListener<io.netty.util.concurrent.Future<? super Integer>>() {
                  @Override
                  public void operationComplete(io.netty.util.concurrent.Future<? super Integer> future) throws Exception {
                      log.info("异步结果：{}",future.getNow());
                  }
              });
  ```

- Promise

  ```java
   public static void main(String[] args) throws ExecutionException, InterruptedException {
              // 1. 创建group
              NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
              EventLoop next = nioEventLoopGroup.next();
              // 2. 主动创建任务容器
              // 说明： 为什么创建时需要绑定一个event loop,原因是为了防止多线程的竞争导致数据错误，其实按照原则来讲promise的所有操作应该都在绑定的event loop中进行
              //       当然也可在具体操作中指定特定的event loop。在此案例中因为只有一个线程在执行放置数据，没有数据竞争的风险，因此不会抛出异常
              DefaultPromise<Integer> integerDefaultPromise = new DefaultPromise<>(next);
              // 3. 创建随意一个线程来在任务容器中放置结果
              new Thread(() -> {
                  log.info("开始计算 。。。 ");
                  try {
                      Thread.sleep(1000);
                      // 4. 成功放置数据
                      integerDefaultPromise.setSuccess(100);
                  } catch (InterruptedException e) {
                      // 4. 失败放置异常
                      integerDefaultPromise.setFailure(e);
                  }
              }).start();
              // 5. 同步获取数据。
              log.info("获取结果：{}", integerDefaultPromise.get());
          }
  ```

###### Handler & Pipeline

