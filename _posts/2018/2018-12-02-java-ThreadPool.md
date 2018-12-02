---
layout: post
title: java 线程池的分析和使用
category: java
tags: [java]
---



##  线程池的分析和使用 

### 为什么要用线程池

* 1，降低资源的消耗，通过重复利用已创建的线程降低线程创建和销毁造成的消耗
* 2，提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
* 3，提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。但是要做到合理的利用线程池，必须要对其了如指掌

### 线程池的使用

#### 线程池的创建

> 我们可以通过ThreadPoolExecutor来创建一个线程池
```
 new ThreadPoolExecutor(corePoolSize, maximumPoolSize,keepAliveTime, milliseconds,runnableTaskQueue, threadFactory,handler);

```

创建一个线程池需要几个参数

* corePoolSize(线程池的基本大小) ： 当提交一个任务到线程池时，线程池户创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池的基本大小就不在创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有的基本线程
* runnableTaskQueue(任务队列)：用于保存等待执行的任务的阻塞队列，可以选择以下几个阻塞队列：
1，ArrayBlockingQueue: 是一个基于数组结构的有界阻塞队列，此队列按FIFO(先进先出)原则对元素进行排序
2，LinkedBlockingQueue: 是一个基于链表结构的阻塞队列，此队列按FIFO(先进先出)排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
3，SynchronousQueue: 一个不存储元素的阻塞队列，每个插入操作必须要等到另一个线程调用移除操作，否则插入一直处于阻塞状态。吞吐量通常高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
4， PriorityBlockingQueue: 一个具有优先级的无限阻塞队列
* maximumPoolSize(线程池最大大小)：线程允许创建的最大线程数。如果队列满了，并且已创建的线程池小于最大线程数，则线程池会在创建新的线程来执行任务。值得注意的是如果使用了无界队列这个参数就没什么效果。
* ThreadFactory: 用于设置创建线程的工厂，可以通过线程工厂给每个线程设置更有意义的名字，Debug和定位问题是非常有帮助
* RejectExecutionHandler(饱和策略) ： 当队列和线程池都满了，说明线程处于饱和状态，那么必须采用一种策略处理提交新的任务。这个策略默认情况下是AbortPolicy,表示无处处理新任务时抛出异常。以下是JDK1.5提供的四种策略。
1，AbortPolicy：直接抛出异常
2，CallerRunsPolicy: 只能调用者所在线程来运行任务。
3，DiscardOldestPolicy: 丢弃队列里最近的一个任务，并执行当前任务
4，DiscardPolicy：不处理，丢弃掉
当然也可以根据应用场景需要来实现RejectExecutionHandler接口自定义策略，如记录日志或持久化不能处理的任务
* keepAliveTime: (线程活动保持时间)：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行比较短，可以调大这个时间，提高线程的利用率。
* TimeUnit(线程活动保持时间的单位)：可选的单位有天（DAYS）,小时(HOURS),分钟(MINUTES),毫秒(MILLISECONDS)，微秒(MICROSECONDS)和毫微秒(NANOSECONDS)




### 