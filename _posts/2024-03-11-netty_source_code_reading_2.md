---
layout: post
title: "Netty源码解读二：NettyServer的启动源码分析"
date: 2024-03-11
tags: [ Netty ]
comments: true
author: zouhuanli
---

本文主要分析NettyServer的启动源码，默认的BossGroup和workerGroup。在mainReactor(线程数量1)+subReactor(线程数量N)的模型中,Netty的bossGroup只有一个EventLoop。

boss的EventLop只处理Accept事件，然后将channel绑定到一个worker中，由一个worker处理全部的IO事件。

说明：EventLop是一个“死循环”的线程，不断轮询IO事件并交给Handler处理。一个Channel只会绑定给一个EventLop处理，一个EventLop由一个线程处理，这样避免了Channel被多个线程处理的并发问题。一个或多个EventLop组成一个EventLoopGroup。



下面具体分析NettyServer的启动源码，理解EventLop的创建过程，以及BossGroup的EventLop和workerGroup的EventLop的处理的不同。

这里直接以EchoServer作为入口分析。其实鉴于Netty对上层的良好透明性，大多时候上层业务基本只要定制ChannelHandler即可，服务器引导类基本相同的。分析EventLoop按默认的NioEventLoop，EpollEventLoop同理的。

本文目录：

- [一.Bootstrap流程](#一bootstrap流程)
  - [1.引导类](#1引导类)
  - [2.handler类](#2handler类)
  - [3.bind流程](#3bind流程)
- [二.ServerBootstrapAcceptor流程](#二serverbootstrapacceptor流程)
- [三.Boss的EventLoop处理流程](#三boss的eventloop处理流程)
- [四.Worker的EventLoop处理流程](#四worker的eventloop处理流程)
- [五. 简单总结](#五-简单总结)
  - [1.BossGroup总结](#1bossgroup总结)
  - [2.WorkerGroup总结](#2workergroup总结)
- [六. 参考资料](#六-参考资料)

# 一.Bootstrap流程
## 1.引导类

Server的引导类代码如下：
```java
  public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx = ServerUtil.buildSslContext();

        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```
这里Boss只负责新连接的Accept事件处理，一般只需要一个线程即可。而worker是负责处理由Boss分派的具体的Channel的IO事件，需要多个线程处理IO事件和调用Handler处理。

这是多Reactor(1个mainReactor+N个subReactor)+多线程(handler)的模型。默认worker的线程数量是2*逻辑核心数量，这里可以看出Netty是设计为IO密集处理。

EventLoopGroup是一个或者一组EventLoop，EventLoop是一个"死循环"线程。

NioEventLoop的run方法如下(有删减)：
```java
 @Override
    protected void run() {
        int selectCnt = 0;
        for (;;) {
            try {
                int strategy;
                                            //获得策略
                                            //这里的策略是若tailTasks不为空，最终调用selectNow立刻返回的轮询操作。
                                            //若tailTasks为空，返回SELECT（-1），执行下面的select(curDeadlineNanos)带超时事件的轮询操作。
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                            //tailTasks为空,select操作
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                    }
             

                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                                //处理SelectionKey
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                                //处理Task
                        // Ensure we always run tasks.
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            
            } 
        }
    

```
可以看到里面的循环主要就是不断调用select获取IO事件，然后processSelectedKeys处理IO事件。

## 2.handler类

再回到Sever代码这里：
```java
 .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });
```
这里添加ChannelHandler，在执行链/管道pipeline添加Handler。一个Worker派发Channel到具体Handler处理器，简单的说Handler是上层具体处理Channel的业务代码。
Inbound和OutBound的方向如下：
```text
 <pre>
 *                                                 I/O Request
 *                                            via {@link Channel} or
 *                                        {@link ChannelHandlerContext}
 *                                                      |
 *  +---------------------------------------------------+---------------+
 *  |                           ChannelPipeline         |               |
 *  |                                                  \|/              |
 *  |    +---------------------+            +-----------+----------+    |
 *  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  .               |
 *  |               .                                   .               |
 *  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
 *  |        [ method call]                       [method call]         |
 *  |               .                                   .               |
 *  |               .                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  +---------------+-----------------------------------+---------------+
 *                  |                                  \|/
 *  +---------------+-----------------------------------+---------------+
 *  |               |                                   |               |
 *  |       [ Socket.read() ]                    [ Socket.write() ]     |
 *  |                                                                   |
 *  |  Netty Internal I/O Threads (Transport Implementation)            |
 *  +-------------------------------------------------------------------+
```

下面进入bind方法，如下：
```java
   // Start the server.
            ChannelFuture f = b.bind(PORT).sync();
```

## 3.bind流程

bind方法具体如下：
```java
 /**
     * Create a new {@link Channel} and bind it.
     */
    public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
    }

    private ChannelFuture doBind(final SocketAddress localAddress) {
                            //创建ServerChannel，和注册ServerChannelServerChannel
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }

                            
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
                                //创建ServerChannel
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
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

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```
具体调试一下这部分代码:

![serverChannel](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/serverChannel.png)

可以看到这里的Channel是ServerChannel。

继续来到init方法：
```java
 @Override
    void init(Channel channel) {
                            //初始化配置
        setChannelOptions(channel, newOptionsArray(), logger);
        setAttributes(channel, newAttributesArray());

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);
        final Collection<ChannelInitializerExtension> extensions = getInitializerExtensions();

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                                            //Acceptor的Handler
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs,
                                extensions));
                    }
                });
            }
        });
                                             //后置处理器
        if (!extensions.isEmpty() && channel instanceof ServerChannel) {
            ServerChannel serverChannel = (ServerChannel) channel;
            for (ChannelInitializerExtension extension : extensions) {
                try {
                    extension.postInitializeServerListenerChannel(serverChannel);
                } catch (Exception e) {
                    logger.warn("Exception thrown from postInitializeServerListenerChannel", e);
                }
            }
        }
    }
```

这里最主要要关注Acceptor的Handler这个Handler对象。

再返回initAndRegister方法：
```java
 ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
```
<strong>这个register方法是将channel注册EventLoop</strong>。

跟踪执行流程，来到AbstractChannel#register方法：
```java

@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
                          //注册Channel
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}

private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
                        //注册Channel。Do方法
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}

```
来到这里AbstractNioChannel#doRegister方法
```java
 @Override
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

这样最终SeverChannel注册到了bossGroup内的EventLoop(只有一个)的Selector上。

再回到doBind0方法：
```java
 private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```
这里其实就是对上面注册结果的回调，如果注册成功，则绑定pipeLine和ChannelHandlerContext。

有一点注意点就是：
```java
 ChannelFuture regFuture = config().group().register(channel);

```
这里因为bossGroup只有一个线程就只能选择一个EventLoop注册。如果是WorkerGroup因为有多个，会调用下面方法选择一个EventLoop注册上去：
```java
@Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
```

到这里bind方法和引导类就解读完成。

下面简单解读ServerBootstrapAcceptor的源码。

# 二.ServerBootstrapAcceptor流程

ServerBootstrapAcceptor最主要的channelRead方法如下:

```java
 @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            setChannelOptions(child, childOptions, logger);
            setAttributes(child, childAttrs);

            if (!extensions.isEmpty()) {
                for (ChannelInitializerExtension extension : extensions) {
                    try {
                        extension.postInitializeServerChildChannel(child);
                    } catch (Exception e) {
                        logger.warn("Exception thrown from postInitializeServerChildChannel", e);
                    }
                }
            }

            try {
                
                //注册clientChannel到worker的EventLoop中
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

其里面最核心的处理就是“ childGroup.register(child).addListener”，注册clientChannel到worker的EventLoop中，也是注册到EventLoop的Selector对象上。
ServerBootstrapAcceptor是InBound的。

具体流程是Boss的EventLoop注册ServerChannel之后，不断监听客户端连接，然后将accept的clientChannel交给ServerBootstrapAcceptor处理，由ServerBootstrapAcceptor将clientChannel
注册到workerGroup的其中一个EventLoop中。

Boss的EventLoop处理流程具体如下。

# 三.Boss的EventLoop处理流程

首先注意Boss绑定的channel是severChannel.

依旧进入EventLoop的run方法:
```java
   int selectNow() throws IOException {
        return selector.selectNow();
    }

```
这里获得事件.

然后处理事件,boss只处理Accept事件.
```java
 private void processSelectedKeys() {
        if (selectedKeys != null) {
            processSelectedKeysOptimized();
        } else {
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```
再进入处理具体事件的代码:
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

这里是Accept事件的处理:

```java
 if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
```

继续来到NioMessageUnsafe#read方法:

```java
 public void read() {
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
            allocHandle.reset(config);

            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                    do {
                                        //执行读取消息操作
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        allocHandle.incMessagesRead(localRead);
                    } while (continueReading(allocHandle));
                } catch (Throwable t) {
                    exception = t;
                }

                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;
                                        //触发channel的read事件的回调/监听器
                    pipeline.fireChannelRead(readBuf.get(i));
                }
                readBuf.clear();
                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (exception != null) {
                    closed = closeOnReadError(exception);

                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    inputShutdown = true;
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```

然后,再来到NioServerSocketChannel:

```java
  @Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }
```

这里就是调用javaChannel(ServerChannel)的accept方法获取客户端连接clientChannel,然后包装为netty的NioSocketChannel.

这里获取clientChannel之后,再进入这行代码,触发channelRead操作的回调:

```java
 pipeline.fireChannelRead(readBuf.get(i));
```

Boss的pipeline如下:

![boss-pipeline](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/boss-pipeline.png)

所以这里会触发回调ServerBootstrapAcceptor的channelRead方法,也就是将clientChannel注册到WorkerGroup的一个EventLoop中.
这行代码注册clientChannel到worker的EventLoop:"childGroup.register(child).addListener(new ChannelFutureListener()"． 调试信息如下:

![clientChannel](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/clientChannel.png)

其内部的pipeline包含EchoServerHandler.

到这里,客户端连接clientChannel就交付给WorkerGroup的一个EventLoop去处理.


# 四.Worker的EventLoop处理流程

Worker的EventLoop处理流程和Boss的一样的,区别是处理的是clientChannel,会按照IO事件具体调用对应ChannelHandler.

这里直接来到processSelectedKey方法:

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

举例,这里依旧是进入READ事件的处理:

```java
   // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
```

只不过,这个不是调用ServerChannel的read方法,而是调用clientChannel的read方法,AbstractNioByteChannel的read方法具体如下:

```java
     @Override
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
                            //分配ByteBuf,clientChannel读取
            byteBuf = allocHandle.allocate(allocator);
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }

            allocHandle.incMessagesRead(1);
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());

        allocHandle.readComplete();
                            //触发读操作事件的回调,也就是ChannelHandler的channelRead方法
        pipeline.fireChannelReadComplete();

        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}

```
![clientChannel-read](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/clientChannel-read.png)


触发ChannelRead回调的源码如下:

```java

    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }

    private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                // DON'T CHANGE
                // Duplex handlers implements both out/in interfaces causing a scalability issue
                // see https://bugs.openjdk.org/browse/JDK-8180450
                final ChannelHandler handler = handler();
                final DefaultChannelPipeline.HeadContext headContext = pipeline.head;
                if (handler == headContext) {
                    headContext.channelRead(this, msg);
                } else if (handler instanceof ChannelDuplexHandler) {
                    ((ChannelDuplexHandler) handler).channelRead(this, msg);
                } else {
                    ((ChannelInboundHandler) handler).channelRead(this, msg);
                }
            } catch (Throwable t) {
                invokeExceptionCaught(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
```

最后就调用我们具体的ChannelHandler,要注意的是Pipeline的调用是链式的,会调用多个ChannelHandler.

![BizHandler](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/BizHandler.png)

到这里,一个客户端请求到我们的业务Handler的流程就基本清晰了.

下面对Boss和worker的EventLoop做下总结.

# 五. 简单总结

## 1.BossGroup总结

1. Boss的EventLoop主要是监听Accept事件,负责不断accept客户端连接.
2. Boss的EventLoop的Selector上注册的是ServerChannel,Boss的EventLoop不断accept监听客户端连接channel,再ServerChannelAcceptor将客户端Channel注册到Worker(s)的一个EventLoop中.

具体如下图:

![BossEventLoop-0](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/BossEventLoop-0.png)


## 2.WorkerGroup总结

worker的EventLoop注册的是clientChannel,不断轮询clientChannel的事件,并处理事件,处理事件时调用Pipeline的链式的ChannelHandler去做具体的处理.

总结如下图:

![WorkerEventLoop-0](https://raw.githubusercontent.com/zouhuanli/zouhuanli.github.io/master/images/2024-03-11-netty_source_code_reading_2/WorkerEventLoop-0.png)



重点要区分Boss是监听serverChannel,不断获取客户端连接clientChannel,并将clientChannel交给一个Worker的EventLoop去处理.

也就是说Boss负责客户端连接事件处理,Worker负责客户端连接的读写事件和具体业务处理.


# 六. 参考资料

1. Netty源码(版本4.1)







