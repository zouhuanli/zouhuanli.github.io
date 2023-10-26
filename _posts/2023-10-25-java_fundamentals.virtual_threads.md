---
layout: post
title: "JEP444解读和虚拟线程初体验"
date: 2023-10-25
tags: [ spring ]
comments: true
author: zouhuanli
---

在2023年9月发布的JDK21版本中，最令人关注的自然是虚拟线程(Virtual Threads)。<br>
JDK21的Release
Notes地址为[https://www.oracle.com/java/technologies/javase/21-relnote-issues.html](https://www.oracle.com/java/technologies/javase/21-relnote-issues.html)。<br>
以及在OpenJDK网站的JEP444-虚拟线程的介绍[https://openjdk.org/jeps/444](https://openjdk.org/jeps/444)。<br>
本文主要解读一下OpenJDK关于虚拟线程介绍的这个文章，以及简单使用一下虚拟线程。建议读者也读一下这篇文章。

本文源码地址为:[https://github.com/zouhuanli/jdk21demo.git](https://github.com/zouhuanli/jdk21demo.git).<br>

# 一、概述

虚拟线程是轻量级线程，可以显着减少编写、维护和观察高吞吐量并发应用程序的工作量。
虚拟线程是由JDK创建和关联，主要用于执行浅调用栈、轻量级操作的任务，不应该被池化，随时创建随时使用随时销毁。
平台线程(JDK21之前的Thread)是和操作系统线程1：1创建，而虚拟线程是M:1创建的，多个虚拟线程对应一个操作系统线程。虚拟线程采用
M:N 调度，其中大量 (M) 虚拟线程被调度在较少数量 (N) 的操作系统线程上运行。
虚拟线程主要是提高吞吐量,原文为“虚拟线程并不是更快的线程——它们运行代码的速度并不比平台线程快。它们的存在是为了提供规模（更高的吞吐量），而不是速度（更低的延迟）。它们的数量可以比平台线程多得多，因此根据利特尔定律，它们可以实现更高吞吐量所需的更高并发性”。
虚拟线程也支持线程本地变量。
值得注意的是虚拟线程也是Thread的一个实例，没有创建新的线程类,通过新的Thread的API来创建虚拟线程。

# 二、基本用法

这里就是原文的示例：
```java
public class NewVirtualThreadPerTaskExecutorDemo {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
        Thread t = Thread.ofVirtual().name("VirtualThread").unstarted(() -> System.out.println("Hello, Virtual Thread!"));
        t.start();
        System.out.println(LocalDateTime.now());
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 10_000).forEach(i -> {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));
                    return i;
                });
            });
        }  // executor.close() is called implicitly, and waits
        System.out.println(LocalDateTime.now());
        try (var executor = Executors.newFixedThreadPool(100)) {
            IntStream.range(0, 10_000).forEach(i -> {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));
                    return i;
                });
            });
        }
        System.out.println(LocalDateTime.now());
        /***
         * Hello, World!
         * 2023-10-25T20:16:24.666452900
         * 2023-10-25T20:16:26.133547
         * 2023-10-25T20:18:07.043543300
         */

    }

}
```
注意看一下上面打印的时间：
```text
 /***
         * Hello, World!
         * 2023-10-25T20:16:24.666452900
         * 2023-10-25T20:16:26.133547
         * 2023-10-25T20:18:07.043543300
         */
```
使用虚拟线程之间2s左右结束了所有的任务，而普通的线程池是每个线程执行10000/100=100个任务，每个任务等待1s，正好是100s左右。
原文的翻译是
```text
如果这个程序使用ExecutorService为每个任务创建一个新的平台线程（例如Executors.newCachedThreadPool(). 将ExecutorService尝试创建 10,000 个平台线程，从而创建 10,000 个操作系统线程，并且程序可能会崩溃，具体取决于计算机和操作系统。

如果程序使用ExecutorService从池中获取平台线程（例如Executors.newFixedThreadPool(200). 这ExecutorService将创建 200 个平台线程，由所有 10,000 个任务共享，因此许多任务将顺序运行而不是并发运行，并且程序将需要很长时间才能完成。对于该程序，具有 200 个平台线程的池只能实现每秒 200 个任务的吞吐量，而虚拟线程可实现每秒约 10,000 个任务的吞吐量（在充分预热后）。此外，如果将10_000示例程序中的 更改为1_000_000，则该程序将提交 1,000,000 个任务，创建 1,000,000 个并发运行的虚拟线程，并且（在充分预热后）实现每秒约 1,000,000 个任务的吞吐量。

如果该程序中的任务执行一秒钟的计算（例如，对一个巨大的数组进行排序），而不是仅仅休眠，那么增加线程数量超出处理器核心数量将无济于事，无论它们是虚拟线程还是平台线程。虚拟线程并不是更快的线程——它们运行代码的速度并不比平台线程快。它们的存在是为了提供规模（更高的吞吐量），而不是速度（更低的延迟）。它们的数量可以比平台线程多得多，因此根据利特尔定律，它们可以实现更高吞吐量所需的更高并发性。

换句话说，虚拟线程可以显着提高应用程序吞吐量当并发任务数较高（数千以上），并且工作负载不受 CPU 限制，因为在这种情况下，线程数多于处理器核心数无法提高吞吐量。
虚拟线程有助于提高典型服务器应用程序的吞吐量，正是因为此类应用程序由大量并发任务组成，而这些任务大部分时间都在等待。
```
看一下其他的基本用法：
```java
        t.setDaemon(true);
        t.start();

        System.out.println(t.isVirtual());
        System.out.println(t.getName());
        System.out.println(t.getState());
        System.out.println(t.getThreadGroup());
        System.out.println(t.getPriority());
```
虚拟线程总是守护线程，虚拟线程有正常5的优先级且无法设置优先级，虚拟线程的Thread.getThreadGroup()返回“VirtualThreads”。<br>
Thread.getAllStackTraces()现在返回所有平台线程的映射，而不是所有线程的映射。

虚拟线程支持支持sync关键字，测试如下：

```java
public class ConcurrentDemo {
    static int num = 0;

    //支持sync关键字
    public static synchronized void addNum() {
        num++;
        System.out.println(Thread.currentThread().getName() + ":" + num);
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 100; i++) {
            Thread t = Thread.ofVirtual().name("VirtualThread" + i).unstarted(ConcurrentDemo::addNum);
            t.start();
        }
        TimeUnit.SECONDS.sleep(5L);
        System.out.println(num);
        //输出：100
    }
}
```

同时虚拟线程支持ThreadLocal。
```java
public class ThreadLocalDemo {
    private static final ThreadLocal<Integer> threadLocal = new ThreadLocal();

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 100; i++) {
            Thread t = Thread.ofVirtual().name("VirtualThread" + i).unstarted(ThreadLocalDemo::addNum);
            t.start();
        }

        TimeUnit.SECONDS.sleep(5L);
    }

    private static void addNum() {
        Integer num = threadLocal.get();
        if (num == null) {
            num = 0;
        }
        num++;
        threadLocal.set(num);
        System.out.println(Thread.currentThread().getName() + ":" + num);
    }
}
```
# 三.阻塞和调度

关于阻塞,看下文章原文：
```text
When code running in a virtual thread calls a blocking I/O operation in the java.* API, the runtime performs a non-blocking OS calland automatically suspends the virtual thread until it can be resumed later.
To Java developers, virtual threads are simply threads that are cheap to create and almost infinitely plentiful.
Hardware utilization is close to optimal, allowing a high level of concurrency and, as a result, high throughput, while the application remains harmonious with the multithreaded design of the Java Platform and its tooling.
```
这里简单翻译一下就是虚拟线程里面的阻塞IO操作会被JDK调度器自动挂起直到后续恢复。


调度是由JDK调度的，而不是由OS执行。依旧是看下原文的翻译：
```text

为了完成有用的工作，需要调度线程，即分配线程在处理器核心上执行。对于作为操作系统线程实现的平台线程，JDK 依赖于操作系统中的调度程序。相比之下，对于虚拟线程，JDK 有自己的调度程序。JDK的调度程序不是直接将虚拟线程分配给处理器，而是将虚拟线程分配给平台线程（这就是前面提到的虚拟线程的M:N调度）。然后，操作系统像往常一样调度平台线程。

JDK的虚拟线程调度程序是一个ForkJoinPool以先进先出（FIFO）模式运行的工作窃取程序。调度程序的并行度是可用于调度虚拟线程的平台线程的数量。默认情况下，它等于可用处理器的数量，但可以通过系统属性进行调整jdk.virtualThreadScheduler.parallelism。这与公共池ForkJoinPool不同，公共池用于例如并行流的实现，并且以 LIFO 模式运行。

调度程序为其分配虚拟线程的平台线程称为虚拟线程的载体。虚拟线程在其生命周期内可以被调度到不同的载体上；换句话说，调度程序不维护虚拟线程和任何特定平台线程之间的关联性。从Java代码的角度来看，一个正在运行的虚拟线程在逻辑上独立于它当前的载体：

虚拟线程无法获取载体的身份标识。Thread.currentThread()返回的值始终是虚拟线程本身。

载体和虚拟线程的堆栈跟踪是分开的。虚拟线程中抛出的异常将不包括载体的堆栈帧。线程转储不会显示虚拟线程堆栈中载体的堆栈帧，反之亦然。

载体的线程局部变量对于虚拟线程不可用，反之亦然。

另外，从Java代码的角度来看，虚拟线程及其载体暂时共享OS线程的事实是不可见的。相比之下，从本机代码的角度来看，虚拟线程及其载体都运行在同一个本机线程上。因此，在同一虚拟线程上多次调用的本机代码可能会在每次调用时观察到不同的操作系统线程标识符。

调度程序当前未实现虚拟线程的时间共享。分时是对消耗了分配的 CPU 时间的线程进行强制抢占。虽然当平台线程数量相对较少且 CPU 利用率为 100% 时，时间共享可以有效减少某些任务的延迟，但尚不清楚时间共享对于 100 万个虚拟线程是否同样有效。
```


简单总结一下：
虚拟线程是轻量级线程，主要提高吞吐量而不是延迟，适合用于IO密集场景，对计算密集CPU密集的任务无明显改善，对于CPU密集、CPU长时间运行的任务增加虚拟线程或者平台线程无帮助。<br>
虚拟线程显然比改善性语法更令人感兴趣，期待虚拟线程为Java语言、第三方框架乃至Java生态带来变革。


# 四、参考材料

JEP444-虚拟线程的介绍[https://openjdk.org/jeps/444](https://openjdk.org/jeps/444)