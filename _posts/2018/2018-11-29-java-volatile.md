---
layout: post
title: 深入解析 volatile 关键字
category: java
tags: [java]
---



## 前言

在Java 5之前，volatile是一个备受争议的关键字，因为在程序中使用它往往会导致出人意料的结果。在Java 5之后，volatile关键字才得以重获生机。

### 内存可见性

由于java内存模型(JMM)规定，所有的变量都存放在主内存中，而每个线程都有自己的工作内存（高速缓存）

线程在工作时，需要将主内存的数据拷贝到工作内存中，这样对数据的任何操作都是基于工作内存（效率提高），并且不能操作主内存以及其他线程工作内存的数据，之后再将更新之后的数据刷新到主内存中。

> 注： 这里所提到的主内存可以认为是堆内存，而工作内存则认为是栈内存。

如下如所示：
![](http://io.dbbaxbb.cn/assets/images/2018/java/volatile1.png) <br/>
所以在并发运行时线程A所读取的数据是线程B更新之前的数据

显然这肯定会出现问题，因此volatile的作用出现了。

* 当一个变量被volatile修饰时，任何线程对它的写操作都会立刻刷新到主内存中，并且会强制让缓存了改变量的线程中的数据清空，必须从主内存重新读取数据。

volatile 修饰之后并不是让线程直接去主内存获取数据， 依然需要将变量拷贝到工作内存中。

#### 内存可见性的应用

当我们需要在两个线程间依据主内存通信时，通信的那个变量就必须用volatile修饰。

```
public class Volatile implements Runnable{
   private static volatile boolean flag = true ;
   @Override
   public void run() {
       while (flag){
           System.out.println(Thread.currentThread().getName() + "正在运行。。。");
       }
       System.out.println(Thread.currentThread().getName() +"执行完毕");
   }
   public static void main(String[] args) throws InterruptedException {
       Volatile aVolatile = new Volatile();
       new Thread(aVolatile,"thread A").start();
       System.out.println("main 线程正在运行") ;
       TimeUnit.MILLISECONDS.sleep(100) ;
       aVolatile.stopThread();
   }
   private void stopThread(){
       flag = false ;
   }
}
```
主线程在修改了标志位使得线程A立即停止，如果没有Volatile修饰，可能会出现延迟。
但这里有个误区，这样的使用方式跟让人感觉是：
> 对Volatile修饰的变量进行并发操作是线程安全的。

重点强调，volatile并不能保证线程安全性！
如下程序：

 ```
 public class VolatileInc implements Runnable{
    private static volatile int count = 0 ; //使用 volatile 修饰基本数据内存不能保证原子性
    //private static AtomicInteger count = new AtomicInteger() ;
    @Override
    public void run() {
        for (int i=0;i<10000 ;i++){
            count ++ ;
            //count.incrementAndGet() ;
        }
    }
    public static void main(String[] args) throws InterruptedException {
        VolatileInc volatileInc = new VolatileInc() ;
        Thread t1 = new Thread(volatileInc,"t1") ;
        Thread t2 = new Thread(volatileInc,"t2") ;
        t1.start();
        //t1.join();
        t2.start();
        //t2.join();
        for (int i=0;i<10000 ;i++){
            count ++ ;
            //count.incrementAndGet();
        }
        System.out.println("最终Count="+count);
    }
 }
 ```
 当我们三个线程（t1,t2,main）同时对int进行累加时会发现最终结果都会小于3000
 
 > 这是因为volatile保证了内存可见性，每个线程拿到的值都是最新值，但是count++这个操作并不是原子的。这里涉及到获取值，自增，赋值等操作并不能同时完成。
 
 * 所以想达到线程安全可以使这三个线程串行执行（其实就是单线程，并没有发挥多线程的优势）
 * 也可以使用synchronize和锁的优势来保证原子性。
 * 还可以用 Atomic 包中 AtomicInteger 来替换 int，它利用了 CAS 算法来保证了原子性。
 
 ### 指令重排
 
 内存可见性只是volatile中的一个语义，还可以放在JVM进行指令重排优化。
 举一个伪代码：
 
  ```
    int a = 10; // 1
    int b = 20; // 2
    int c = a + b; // 3
  ```
一段简短的代码，理想的情况下他的执行顺序应该是1 > 2 > 3。但可能进过JVM优化之后变成2 > 1 > 3.
 
可以发现不管JVM怎么优化，前提都是保证单线程结果不变的情况下进行的。
 
可能这样还看不出什么问题，结合下面一段伪代码：

 ```
 private static Map<String,String> value ;
 private static volatile boolean flag = fasle ;
 //以下方法发生在线程 A 中 初始化 Map
 public void initMap(){
    //耗时操作
    value = getMapValue() ;//1
    flag = true ;//2
 }
 //发生在线程 B中 等到 Map 初始化成功进行其他操作
 public void doSomeThing(){
    while(!flag){
        sleep() ;
    }
    //dosomething
    doSomeThing(value);
 }
 ```
 这里就可以看出问题了，当flag没有被volatile修饰时，JVM对1和2进行重排，导致value都还没有初始化可能就被B占用了。
 所以加上volatile之后可以防止这样的重排优化，保证业务正确性。
 #### 指令重排的应用
 
 一个经典的使用场景就是双重懒加载的单例模式了:
 
  ```
  public class Singleton {
     private static volatile Singleton singleton;
     private Singleton() {
     }
     public static Singleton getInstance() {
         if (singleton == null) {
             synchronized (Singleton.class) {
                 if (singleton == null) {
                     //防止指令重排
                     singleton = new Singleton();
                 }
             }
         }
         return singleton;
     }
  }
  ```
 这里的volatile关键字主要是为了防止指令重排
 如果不用，singleton=new Singleton();，这段代码其实是分为三步：
 * 分配内存空间
 * 初始化对象
 * 将singleton对象指向分配的地址
加上 volatile 是为了让以上的三步操作顺序执行，反之有可能第二步在第三步之前被执行就有可能某个线程拿到的单例对象是还没有初始化的，以致于报错

总结
volatile 在 Java 并发中用的很多，比如像 Atomic 包中的 value、以及 AbstractQueuedLongSynchronizer中的 state 都是被定义为 volatile 来用于保证内存可见性。

将这块理解透彻对我们编写并发程序时可以提供很大帮助。

[参考链接](http://ifeve.com/%E4%BD%A0%E5%BA%94%E8%AF%A5%E7%9F%A5%E9%81%93%E7%9A%84-volatile-%E5%85%B3%E9%94%AE%E5%AD%97/#more-37338)