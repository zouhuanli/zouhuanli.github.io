---
layout: post
title: "Netty源码解读:总览和系列计划"
date: 2024-01-28
tags: [ Netty ]
comments: true
author: zouhuanli
---

本系列是新开的解读Netty源码的文章系列。

要解读Netty源码需要做比较多的前期铺垫内容，主要是Linux网络、Java NIO的内容，特别是Reactor模型,这些基础内容对理解其他网络框架也是有所帮助的。

本系列文章在网络模型、Java NIO的基础之上，打算按照两条脉络来解读Netty的使用和源码分析：<br>
1. 组件的创建、启动、初始化和启动。<br>
2. 网络请求的收发处理过程。<br>

本系列文章主要参考资料有：<br>
1. 《深入理解Linux网络技术内幕》<br>
2. 《计算机网络：自顶向下方法》<br>
3. 《TCP/IP协议详解》<br>
4. 《HTTP权威指南》<br>
5. 《Netty实战》<br>

# 一、NIO基础
在介绍Netty之前需要了解一些Java NIO的基础知识，Java NIO提供了不同于BIO的IO模型。

普通的阻塞读，对通道发起Read操作会阻塞当前线程等待操作系统准备数据就绪，再从内核缓存区复制到进程缓存区，完成读取操作。

NIO的同步非阻塞读，或者说多路复用IO，使用复用器器selector轮询通道的读就绪/写就绪事件，通道可读/可写在由内核缓存区复制到进程缓存区，完成读取操作。

需要注意的是NIO不是真正的异步非阻塞读，异步非阻塞IO或者说AIO是JDK1.7新加入的AsyncXXXChannel等类型，实现了真正的异步非阻塞读/写、

下面是一些BIO、NIO、AIO的实际例子，帮助我们理解和比较三者。案例来源是李林峰老师的《Netty权威指南》这本书。

源码地址是：https://github.com/zouhuanli/NettyTheDefinitiveGuide.git。

## 1.1 BIO案例

一般认为，BIO是最简单，并发性能最差的IO类型。只适合很简单，性能要求不高的网络编程需求。

案例的基本功能是client发生Query命令，server获取当前时间并返回给client。案例仅供学习，未处理异常和优化结构。

BIO的Server源码如下：
```java
public class TimeServer {

    public static void  main(String [] args) throws IOException {
        int  port = 8080;
        ServerSocket server;
        try {
            server = new ServerSocket(port);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        System.out.println("The time server is start in port : " + port);
        while(true){
            new Thread(new TimeServerHandler(server.accept())).start();
        }
    }
}

```
```java
public class TimeServerHandler implements Runnable {
    private final Socket clientSocket;

    public TimeServerHandler(Socket clientSocket) {
        this.clientSocket = clientSocket;
    }


    @Override
    public void run() {
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            out = new PrintWriter(clientSocket.getOutputStream(), true);
            String currentTime;
            String body;
            while (true) {
                body = in.readLine();

                if (body == null) {
                    break;
                }
                System.out.println("The time server receive order : " + body);
                currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) : "BAD ORDER";
                out.println(currentTime);
            }
        } catch (IOException e) {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException ex) {
                    ex.printStackTrace();
                }
            }
            if (out != null) {
                out.close();
            }
            try {
                this.clientSocket.close();
            } catch (IOException ex) {
                throw new RuntimeException(ex);
            }

        }
        if (in != null) {
            try {
                in.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (out != null) {
            out.close();
        }
    }
}
```

基本操作就是通过ServerSocket.accept接受客户端连接，并写回当前时间。编码模型还是很简单的。

客户端源码如下：
```java
public class TimeClient {
    /**
     * 读取控制台输入 发送到127.0.0.1:8080，反现服务器返回结果
     * BAD PRACTICE:未处理异常
     *
     * @param args
     * @throws IOException
     */
    public static void main(String[] args) throws IOException {
        int port = 8080;
        Socket clientSocket;
        BufferedReader in;
        PrintWriter out;
        clientSocket = new Socket("127.0.0.1", port);
        in = new BufferedReader(new java.io.InputStreamReader(clientSocket.getInputStream()));
        out = new PrintWriter(clientSocket.getOutputStream(), true);
        out.println("QUERY TIME ORDER");
        System.out.println("Send order 2 server succeed.");
        String resp = in.readLine();
        System.out.println("Now is :" + resp);
    }
}
```

可以看到BIO的Server和Client的编码模型是很简单，适合并发量很小、无性能要求的场景。

## 1.2 NIO案例

NIO的核心原理是通过Reactor模式的轮询器/复用器Selector不断轮询注册的Channel事件，然后再交给具体线程去处理通道的读/写。具体的通道内容的读/写，这块是阻塞的，不是异步直接获取结果的。

NIO的底层是封装Epoll，本质是Linux的epoll的事件轮询。

下面来看下Java原生的NIO的Server和Client源码案例。

```java
public class TimeServer {

    public static void main(String[] args) {
        int port = 8080;
        NioTimeServerHandler server;
        server = new NioTimeServerHandler(port);
        new Thread(server).start();
    }
}

```
```java
public class NioTimeServerHandler implements Runnable {
    private final int port;
    private final Selector selector;
    private final ServerSocketChannel serverSocketChannel;
    private volatile boolean stop;

    public NioTimeServerHandler(int port) {
        this.port = port;
        try {
            selector = Selector.open();
            serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().bind(new InetSocketAddress(this.port), 1024);
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("The time server is start in port : " + port);
            stop = false;
        } catch (ClosedChannelException e) {
            throw new RuntimeException(e);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public void stop() {
        stop = true;
    }

    @Override
    public void run() {
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    }
                }
            } catch (IOException e) {
                try {
                    selector.close();
                } catch (IOException ex) {
                    ex.printStackTrace();

                }
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            // 处理新接入的请求消息
            if (key.isAcceptable()) {
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                try {
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    sc.register(selector, SelectionKey.OP_READ);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            // 处理读消息
            if (key.isReadable()) {
                SocketChannel socketChannel = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = 0;
                readBytes = socketChannel.read(readBuffer);

                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = "";
                    body = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("The time server receive order : " + body);
                    String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) : "BAD ORDER";
                    doWrite(socketChannel, currentTime);
                } else if (readBytes < 0) {
                    key.cancel();
                    try {
                        socketChannel.close();
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                } else {
                    // 读到0字节，忽略
                }


            }
        }
    }

    // 异步写
    private void doWrite(SocketChannel socketChannel, String currentTime) {
        if (currentTime != null && currentTime.trim().length() > 0) {
            ByteBuffer writeBuffer = ByteBuffer.allocate(currentTime.getBytes().length);
            writeBuffer.put(currentTime.getBytes());
            writeBuffer.flip();
            try {
                socketChannel.write(writeBuffer);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```
我们先看下Server的代码。上面源码可能没有处理粘包/半包情况，客户端多次调用可能会有错误。

Server主要流程是创建ServerChannel，bind端口，配置非阻塞。然后打开Selector，ServerChannel注册到Selector上注册Accept事件。

再"死循环"轮询Selector获得Channel的事件：如果是ServerChannel的accept事件，则调用ServerChannel开始接受客户端连接clientChannel,并设置clientChannel为非阻塞和注册它的READ事件。

然后轮询clientChannel的READ/WRITE事件，再交给具体的线程或者Handler去处理。

可以看到整体的编程模型是比BIO复杂了很多。

回到NIO的Client：
```java
public class NioTimeClient {
    public static void main(String[] args) throws IOException
    {
        int port = 8080;
        if (args != null && args.length > 0)
        {
            try
            {
                port = Integer.valueOf(args[0]);
            }
            catch (NumberFormatException e)
            {
                // 采用默认值
            }
        }
        new Thread(new NioTimeClientHandler("127.0.0.1", port), "NIO-TimeClient-001").start();
    }
}
```
```java

public class NioTimeClientHandler implements Runnable {

    private final int port;
    private final String host;
    private Selector selector;
    private SocketChannel socketChannel;
    private volatile boolean stop;

    public NioTimeClientHandler(String host, int port) {
        this.host = host;
        this.port = port;
        stop = false;
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }


    @Override
    public void run() {
        try {
            doConnect();
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
        while (!stop) {
            try {
                selector.select(1000);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> it = selectedKeys.iterator();
            SelectionKey selectionKey;
            while (it.hasNext()) {
                selectionKey = it.next();
                it.remove();
                try {
                    handleInput(selectionKey);
                } catch (Exception e) {
                    if (selectionKey != null) {
                        selectionKey.cancel();
                        if (selectionKey.channel() != null) {
                            try {
                                selectionKey.channel().close();
                            } catch (IOException ex) {
                                throw new RuntimeException(ex);
                            }
                        }
                    }
                }
            }

        }

    }

    private void handleInput(SelectionKey selectionKey) throws IOException {
        if (selectionKey.isValid()) {
            SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
            // 连接成功
            if (selectionKey.isConnectable()) {
                if (socketChannel.finishConnect()) {
                    socketChannel.register(selector, SelectionKey.OP_READ);
                    doWrite(socketChannel);
                } else {
                    System.exit(1);
                }
            }
            if (selectionKey.isReadable()) {
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = socketChannel.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("Now is : " + body);
                    this.stop = true;
                } else if (readBytes < 0) {
                    selectionKey.cancel();
                    socketChannel.close();
                }
            }
        }

    }

    private void doConnect() throws IOException {
        if (socketChannel.connect(new InetSocketAddress(host, port))) {
            socketChannel.register(selector, SelectionKey.OP_READ);
            doWrite(socketChannel);
        } else {
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
    }

    private void doWrite(SocketChannel socketChannel) {
        try {
            String req = "QUERY TIME ORDER";
            byte[] bytes = req.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            socketChannel.write(writeBuffer);
            if (!writeBuffer.hasRemaining()) {
                System.out.println("Send order 2 server succeed.");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
这里就不做过多解读了。其主要流程是clientChannel先执行connect，再轮询监听READ事件，再做对于处理。


## 1.3 AIO案例

NIO的核心是Selector或者说一个单独的线程不断轮询获取Epoll上注册的IO事件，获得就绪事件，在就绪读/就绪写时候交给具体的线程或者Handler处理。其在等待就绪解读是非阻塞的，但是仍是由进程或者上层的业务代码执行Channel的READ操作，这步还是阻塞的。

而AIO是完全异步的，进程或者上层的业务代码发起Channel的读/写后，完全不关心何时就绪，也不需要其主动发起读/写。其操作都交给OS执行，OS读/写完成后再通过回调给进程/业务代码回调结果。

从这里看，理论上,AIO会比NIO有更高的性能和吞吐。实际上，由于Linux AIO支持不完善，Java AIO的编程复杂以及缺少完善的支持，目前Netty还是使用多路复用NIO的。Tomcat在比较新的版本是有AIO实现的，在其Nio2EndPoint内。

因为AIO主要编程工作就是在回调的处理，因此编程复杂度会比NIO更低。

下面来看下AIO的简单案例。

Server案例源码：
```java
public class TimeServer {
    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                // 采用默认值
            }
        }
        AioTimeSeverHandler timeServer = new AioTimeSeverHandler(port);
        new Thread(timeServer, "AIO-AsyncTimeServerHandler-001").start();
    }
}
```
```java
public class AioTimeSeverHandler implements Runnable {
    private int port;
    private CountDownLatch latch;
    private AsynchronousServerSocketChannel serverSocketChannel;

    public AioTimeSeverHandler(int port) {
        this.port = port;
        try {
            serverSocketChannel = AsynchronousServerSocketChannel.open();
            serverSocketChannel.bind(new InetSocketAddress(port));
            System.out.println("The time server is start in port : " + port);

        } catch (Exception e) {
            e.printStackTrace();
            System.exit(-1);
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);
        serverSocketChannel.accept(this, new AcceptCompletionHandler());
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, AioTimeSeverHandler> {
        @Override
        public void completed(AsynchronousSocketChannel result, AioTimeSeverHandler attachment) {
            attachment.serverSocketChannel.accept(attachment, this);
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            result.read(buffer, buffer, new ReadCompletionHandler(result));
        }

        @Override
        public void failed(Throwable exc, AioTimeSeverHandler attachment) {
            attachment.latch.countDown();
        }
    }

    private static class ReadCompletionHandler implements CompletionHandler<Integer, ByteBuffer> {
        private AsynchronousSocketChannel channel;

        public ReadCompletionHandler(AsynchronousSocketChannel channel) {
            this.channel = channel;
        }

        @Override
        public void completed(Integer result, ByteBuffer attachment) {
            attachment.flip();
            byte[] bytes = new byte[attachment.remaining()];
            attachment.get(bytes);
            String body = new String(bytes, StandardCharsets.UTF_8);
            System.out.println("The time server receive order : " + body);
            String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new java.util.Date())
                    : "BAD ORDER";
            doWrite(currentTime);
        }

        private void doWrite(String currentTime) {
            if (currentTime != null && !currentTime.trim().isEmpty()) {
                byte[] bytes = currentTime.getBytes();
                ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
                writeBuffer.put(bytes);
                writeBuffer.flip();
                channel.write(writeBuffer, writeBuffer, new CompletionHandler<>() {
                    @Override
                    public void completed(Integer result, ByteBuffer attachment) {
                        if (attachment.hasRemaining()) {
                            channel.write(attachment, attachment, this);
                        }
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        try {
                            channel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                });
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            try {
                this.channel.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```
可以看到Server的源码主要是打开AsyncChannel，然后注册回调函数。其主要编程工作就是回调函数。

Client端的源码如下：
```java
public class TimeClient {
    public static void main(String[] args) {
        new Thread(new TimeClientHandler("127.0.0.1", 8080)).start();
    }
}
```
```java

public class TimeClientHandler implements Runnable, CompletionHandler<Void, TimeClientHandler> {
    private AsynchronousSocketChannel client;
    private final String host;
    private final int port;
    private CountDownLatch latch;

    public TimeClientHandler(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            client = AsynchronousSocketChannel.open();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);
        client.connect(new InetSocketAddress(host, port), this, this);
        try {
            latch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
        try {
            client.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    @Override
    public void completed(Void result, TimeClientHandler attachment) {
        System.out.println("客户端连接成功");
        byte[] req = "QUERY TIME ORDER".getBytes();

        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        client.write(writeBuffer, writeBuffer, new CompletionHandler<>() {
            @Override
            public void completed(Integer result, ByteBuffer attachment) {
                if (attachment.hasRemaining()) {
                    client.write(attachment, attachment, this);
                } else {
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    client.read(readBuffer, readBuffer, new CompletionHandler<>() {
                        @Override
                        public void completed(Integer result, ByteBuffer attachment) {
                            attachment.flip();
                            byte[] bytes = new byte[attachment.remaining()];
                            attachment.get(bytes);
                            String body;
                            body = new String(bytes, StandardCharsets.UTF_8);
                            System.out.println("Now is :" + body);
                            latch.countDown();
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer attachment) {
                            try {
                                client.close();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    });
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    client.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                latch.countDown();
            }
        });

    }

    @Override
    public void failed(Throwable exc, TimeClientHandler attachment) {

    }
}

```

为避免过多匿名内部类的回调地狱，可以设计好单独的回调类。


# 二、Netty简述

Netty是一个高性能、可伸缩、上层易用的Java网络通信框架，主要用于构建Client/Server应用。其基于异步事件驱动，通过事件和回调来处理网络事件，最大程度避免阻塞对性能的影响。

同时提供了简单良好易用的API，封装或者完善Java的原生NIO，避免面对直接使用Java的原生NIO的复杂度，专注于上层业务开发，减少对底层网络通讯细节的关注。其对上层开放的接口主要ChannelHandler，ChannelPipeline，ChannelContext等服务编排类。

此外，netty在很多开源框架/组件中都有应用，充当底层的网络通信模块，如Dubbo的NettyClient和NettyServer。

总结的说，Netty是一个高性能的Java网络通信框架,有着广泛的应用。


 
# 三、Netty入门

使用Netty改写上面的Client和Server的案例如下。

## 3.1 Server侧源码

Server源码：
```java
/**
 * 引导类和服务端，比较固化
 */
public class TimeServer {
    public static void main(String[] args) {
        int port = 8080;
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChildChannelHandler())
            ;
            ChannelFuture future;
            future = bootstrap.bind(port).sync();
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            System.exit(-1);
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private static class ChildChannelHandler extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel ch) {
            //主要拓展点
            ch.pipeline().addLast(new TimeServerHandler());

        }
    }
}
```
引导类是比较固化的，Server侧需要创建两个EventLoopGroup，一个用于接受客户端连接，一个用于处理客户端的读写事件。EventLoopGroup类似与ThreadGroup，内部的EventLoop是一个轮询线程，轮询注册事件的状态然后进行处理。

bossGroup内部一般是一个EventLoop，也就是只需要一个轮询线程ServeChannel的事件并不断accept客户端连接。worker的EventLoopGroup内部一般是逻辑CPU个数个EventLoop，轮询处理客户端的读写事件。

对上层开发者的主要拓展点是ChannelHandler。

Server的Handler如下：
```java
public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, StandardCharsets.UTF_8);
        System.out.println("The time server receive order : " + body);
        String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) : "BAD ORDER";
        ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes(StandardCharsets.UTF_8));
        ctx.write(resp);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }

}
```
handler主要是实现Channel事件的回调。

## 3.2 Client侧源码

Client侧源码如下：
```java
/**
 * 引导类和客户端，比较固化
 */
public class TimeClient {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new ChannelInitializer() {
                             @Override
                             protected void initChannel(Channel ch) {
                                 //主要拓展点
                                 ch.pipeline().addLast(new TimeClientHandler());
                             }
                         }
                );
        try {
            ChannelFuture future = bootstrap.connect("127.0.0.1", 8080).sync();
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            System.exit(-1);
        } finally {
            group.shutdownGracefully();
        }
    }
}

```
```java

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private final ByteBuf firstMessage;

    public TimeClientHandler() {
        byte[] req = "QUERY TIME ORDER".getBytes();
        firstMessage = Unpooled.copiedBuffer(req);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(firstMessage);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, StandardCharsets.UTF_8);
        System.out.println("Now is : " + body);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // 释放资源
        System.out.println("Unexpected exception from downstream : " + cause.getMessage());
        ctx.close();
    }
}

```
类似于Server代码，不过Client只需要一个EventLoopGroup。同样上层开发者主要开发和实现ChannelHandler的回调方法。

和上面原生Java NIO代码相比，Netty的引导类比较固定，主要是开发ChannelHandler。Netty的代码结构更加清晰，代码量也更少。


# 四、Netty核心架构

## 4.1 组件架构
首先看下官网的系统组件图：
![components](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-01-28-netty_source_code_reading_1/components.png)

整体系统组件分为三层：
1. Core 核心层：事件模型、通用网络API、ByteBuf等。
2. Transport 传输层：TCP、UDP、Socket等网络传输能力的抽象和封装。
3. Protocol 协议层：主要是Http、Websocket、Protobuf等协议的官方实现。

## 4.2 逻辑架构

上图的组件结构重点在于组件功能，划分的结构缺少了组件之间的交互和关联。笔者更喜欢划分成这样的三层架构(图片来源：阿里云开发者公众号)：

![netty-arch](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-01-28-netty_source_code_reading_1/netty-arch.png)

这样的划分不仅仅区分组件的功能，还表明组件的依赖关系和交互关系。

1. 首先是最底层的网络通信层：封装和优化原生Java NIO。核心组件是Bootstrap、Channel、ServerBootstrap。引导类是服务器/客户端启动初始化、服务器连接等操作的核心类。

Channel是对Java NIO的Channel的更高抽象，而且不仅仅包含NIO的channel， 也支持BIO的channel。

2. 然后是调度层：或者说是核心引擎层，负责线程池、事件循环、事件分发等。核心组件是EventLoopGroup、EventLoop等。其最主要的组件就是EventLoop，它是Netty的精髓所在，负责事件分发、线程池调度等。

Server侧的NioEventLoop实现了reactor模式，其boss线程负责accept客户端连接，worker线程负责处理客户端读写事件。

3. 最后是服务编排层:负责组装服务，链式传递服务。核心组件是ChannelPipeline、ChannelHandler、ChannelHandlerContext等。ChannelHandler是上层开发者主要拓展点，主要负责实现Channel事件的回调。


# 五、核心源码总览

上面提及了Netty的一些核心类，如：Bootstrap、Channel、ServerBootstrap、EventLoopGroup、EventLoop、ChannelPipeline、ChannelHandler、ChannelHandlerContext等。

本文不会对这些核心类做过多的解读，而是简单介绍一下Server执行引擎(EventLoop)是如何串联或者使用这些核心类的。

主要是要区分Server的bossEventLoop和workerEventLoop的执行流程。

## 5.1 Server启动和引导过程源码简单解读

首先Server启动的两个EventLoopGroup中，BossGroup是1线程的，主要是使用ServerChannel不断获取客户端连接clientChannel。将clientChannel注册和分配到workerEventLoop(默认2*CPU数量)中，会从workerEventLoopGroup中获得一个注册进去。

然后有workerEventLoop的去处理这个客户端连接channel的IO事件。这是二者最主要的区别。

启动过程中首先需要创建ServerChannel，和注册ServerChannelServerChannel。
```java
 final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
                                //创建ServerChannel
            channel = channelFactory.newChannel();
            init(channel);
        } 
                                 //注册ServerChannel
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
```
然后注册Channel到Selector。Selector是eventLoop（一个EventLoop一般一个线程）里面只有一个。
```java
protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                                    //注册Channel到Selector。Selector是eventLoop（一个线程）里面只有一个。
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```
最后使用ServerBootstrapAcceptor注册clientChannel到worker的EventLoop中。

具体的执行流程大致如下：

## 5.2 bossEventLoop执行过程源码简单解读

在BossEventLoop的run方法中，首先不断轮询Epoll事件，获得ACCEPT事件。BossEventLoop再处理ACCEPT事件。
```java
int selectNow() throws IOException {
        return selector.selectNow();
    }
```

对ACCEPT事件处理是从这里开始：
```java
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
```

来到：
```java
doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());
```
最终获得客户端连接clientChannel。

然后注册clientChannel到workerEventLoop中。然后回调ServerBootstrapAcceptor的channelRead方法,也就是将clientChannel注册到WorkerGroup的一个EventLoop中.
```java
childGroup.register(child).addListener(new ChannelFutureListener();
```

这样bossEventLoop就处理完一个客户端连接，将其注册和交付给workerEventLoop中的一个EventLoop(循环线程)。

下面简单介绍workerEventLoop的流程：

## 5.3 workerEventLoop执行流程源码简单解读

在workerEventLoop的run方法中，具体处理clientChannel的READ或者WRITE事件。
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                // If the channel implementation throws an exception because there is no event loop, we ignore this
                // because we are only trying to determine if ch is registered to this event loop and thus has authority
                // to close ch.
                return;
            }
            // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
            // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
            // still healthy and should not be closed.
            // See https://github.com/netty/netty/issues/5125
            if (eventLoop == this) {
                // close the channel if the key is not valid anymore
                unsafe.close(unsafe.voidPromise());
            }
            return;
        }

        try {
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
               unsafe.forceFlush();
            }

            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```
一个workerEventLoop只有一个Selector，可以处理不同个clientChannel的READ或者WRITE事件。但是一个clientChannel只会被注册和分配到workerEventLoop的一个EventLoop中。

处理具体的READ和WRITE事件时候，会回调ChannelHandler注册的回调方法。通过Pipeline的链式调用,会调用多个ChannelHandler。ChannelHandler就是上层开发者具体开发的业务功能点或者拓展点。


# 六、总结

1. Netty是什么？

    Netty是一个高性能的异步事件驱动的网络应用框架，封装和优化Java NIO，极大的简化网络应用开发的难度，方便我们开发CS应用和私有协议。

2. Netty整体架构

 1） 网络通信层：封装和优化原生Java NIO。提供Bootstrap、Channel、ServerBootstrap等类，负责服务器/客户端启动初始化、服务器连接等功能。这层整体还是优化Java NIO。

 2） 调度引擎层：提供EventLoopGroup、EventLoop等，负责线程池、事件循环、事件分发等功能。这层是实际Netty线程执行事件循环的层。

 3）服务编排层:负责组装服务，链式传递服务。提供ChannelPipeline、ChannelHandler、ChannelHandlerContext等。ChannelHandler是上层开发者主要拓展点，主要负责实现Channel事件的回调。这层是上层开发拓展业务的层。

3. Server执行核心流程：

   Boss的EventLoop主要是监听Accept事件,负责ACCEPT事件，不断accept客户端连接，然后注册和交付给workerEventLoop中的一个EventLoop。boss一般是只有一个线程。

   worker的EventLoop不断轮询clientChannel的IO事件,并处理IO事件,处理事件时调用Pipeline的链式的ChannelHandler去做具体的处理。worker一般是多个线程。

   这是MainRector+多SubReactor的模式。

# 七、参考材料

1. 《Netty权威指南》

2. Netty源码(4.1)






