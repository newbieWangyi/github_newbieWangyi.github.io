---
layout: post
title: 基于redis的分布式锁
category: tool
tags: [tool]
---



## 基于redis的分布式锁

### 1,介绍
这篇博文讲介绍如何一步步构建一个基于Redis的分布式锁。会从最原始的版本开始，然后根据问题进行调整，最后完成一个较为合理的分布式锁。

本篇文章会将分布式锁的实现分为两部分，一个是单机环境，另一个是集群环境下的Redis锁实现。在介绍分布式锁的实现之前，先来了解下分布式锁的一些信息。


### 2，分布式锁

#### 2.1 什么是分布式锁
分布式锁是控制分布式系统或不同系统之间共同访问共享资源的一种锁实现，如果不同的系统或同一个系统的不同主机之间共享了某个资源时，往往需要互斥来防止彼此干扰来保证一致性。



#### 2.2 分布式锁需要具备哪些条件
1, 互斥性：在任意时刻，只有一个客户端持有锁
2，无死锁：即便持有锁的客户端崩溃或者其它意外事件，锁任然可以被获取
3，容错：只要大部分Redis节点都活着，客户端就可以获取和释放锁

#### 2.3 分布式锁的常用实现方式
1，数据库
2，Memcached（add命令）
3, Redis(setNX命令)
4，Zookeeper(临时节点)

### 3，单机Redis的分布式锁

#### 3.1 准备工作

##### 3.1.1 定义常量类
 ```
 public class LockConstants {
     public static final String OK = "OK";
 
     /** NX|XX, NX -- Only set the key if it does not already exist. XX -- Only set the key if it already exist. **/
     public static final String NOT_EXIST = "NX";
     public static final String EXIST = "XX";
 
     /** expx EX|PX, expire time units: EX = seconds; PX = milliseconds **/
     public static final String SECONDS = "EX";
     public static final String MILLISECONDS = "PX";
 
     private LockConstants() {}
 }
 ```
##### 3.1.2 定义锁的抽象类
抽象类RedisLock实现java.util.concurrent包下的Lock接口，然后对一些方法提供默认实现，子类只需实现lock方法和unlock方法即可。代码如下
 ```
public abstract class RedisLock implements Lock {
 
     protected Jedis jedis;
     protected String lockKey;
 
     public RedisLock(Jedis jedis,String lockKey) {
         this(jedis, lockKey);
     }
 
 
     public void sleepBySencond(int sencond){
         try {
             Thread.sleep(sencond*1000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
     }
 
 
     @Override
     public void lockInterruptibly(){}
 
     @Override
     public Condition newCondition() {
         return null;
     }
 
     @Override
     public boolean tryLock() {
         return false;
     }
 
     @Override
     public boolean tryLock(long time, TimeUnit unit){
         return false;
     }
 }
 ```

#### 3.2 各版本如下，先从最基础版本开始

##### 3.2.1代码如下

 ```
 public class LockCase1 extends RedisLock {
 
     public LockCase1(Jedis jedis, String name) {
         super(jedis, name);
     }
 
     @Override
     public void lock() {
         while(true){
             String result = jedis.set(lockKey, "value", NOT_EXIST);
             if(OK.equals(result)){
                 System.out.println(Thread.currentThread().getId()+"加锁成功!");
                 break;
             }
         }
     }
 
     @Override
     public void unlock() {
         jedis.del(lockKey);
     }
 }
 ```
LockCase1类提供了lock和unlock方法，其中lock方法也就是在redis客户端执行如下命令

 ```
 SET lockKey value NX
 ```
而unlock方法就是调用DEL命令将键删除

但仔细想一下，这个功能会出现什么问题呢
假如有两个客户端A和B,A获得分布式的锁。A执行了一会，突然A所在的服务器断电了（或者其他什么），也就是客户端A挂了，这是出现了一个问题，这个锁一直存在，且不会被释放，其他客户端永远也获不得锁。
![](http://io.dbbaxbb.cn/assets/images/2018/docker/redisLock.png) <br/>

可以通过设置过期时间来解决这个问题


##### 3.2.2 设置锁的过期时间

 ``` 
 public void lock() {
     while(true){
         String result = jedis.set(lockKey, "value", NOT_EXIST,SECONDS,30);
         if(OK.equals(result)){
             System.out.println(Thread.currentThread().getId()+"加锁成功!");
             break;
         }
     }
 }
 ```
类似的redis命令

 ```
 SET lockKey value NX EX 30
 ```
* 注：要保证设置过期时间和设置锁具有原子性

这时又出现了另外一个问题，如下
* 客户端A获得锁过期时间30秒
* 客户端A在操作是阻塞了50秒
* 30秒到了，锁自动释放
* 客户端B获取到对应的同一个资源的锁
* 客户端A从阻塞中恢复过来，释放掉客户端B持有的锁
示意图如下：
![](http://io.dbbaxbb.cn/assets/images/2018/docker/redisLock1.png) <br/>

这时会有两个问题
1，过期时间如果保证大于业务时间
2，如何保证锁不会被误删除

先来解决如果保证锁不会被误删除这个问题
这个问题可以通过设置value为客户端生成的一个随机字符串，且保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。

##### 3.2.3 设置锁的value

抽象类RedisLock增加lockValue字段，lockValue字段的默认值为UUID随机值假加上当前线程

 ```
 public abstract class RedisLock implements Lock {
 
     //...
     protected String lockValue;
 
     public RedisLock(Jedis jedis,String lockKey) {
         this(jedis, lockKey, UUID.randomUUID().toString()+Thread.currentThread().getId());
     }
 
     public RedisLock(Jedis jedis, String lockKey, String lockValue) {
         this.jedis = jedis;
         this.lockKey = lockKey;
         this.lockValue = lockValue;
     }
 
     //...
 }
 ```

加锁代码

 ```
 public void lock() {
     while(true){
         String result = jedis.set(lockKey, lockValue, NOT_EXIST,SECONDS,30);
         if(OK.equals(result)){
             System.out.println(Thread.currentThread().getId()+"加锁成功!");
             break;
         }
     }
 }
 ```
 解锁代码
 
  ```
  public void unlock() {
      String lockValue = jedis.get(lockKey);
      if (lockValue.equals(lockValue)){
          jedis.del(lockKey);
      }
  }
  ```
这时看看加锁代码，好像没有什么问题啊。再来看看解锁的代码，这里的解锁操作包含三步操作：获取值、判断和删除锁。这时你有没有想到在多线程环境下的i++操作?

##### 3.2.4 i++问题
i++操作也可分为三个步骤：读i的值，进行i+1，设置i的值。
如果两个线程同时对i进行i++操作，会出现如下情况

1，i设置值为0
2，线程A读到i的值为0
3，线程B也读到i的值为0
4，线程A执行了+1操作，将结果值1写入到内存
5，线程B执行了+1操作，将结果值1写入到内存
6，此时i进行了两次i++操作，但是结果却为1

在多线程环境下有什么方式可以避免这类情况发生?
解决方式有很多种，例如用AtomicInteger、CAS、synchronized等等。
这些解决方式的目的都是要确保i++ 操作的原子性。那么回过头来看看解锁，同理我们也是要确保解锁的原子性。我们可以利用Redis的lua脚本来实现解锁操作的原子性。
##### 3.2.5 具有原子性的释放锁
lua脚本内容如下

 ```
 if redis.call("get",KEYS[1]) == ARGV[1] then
     return redis.call("del",KEYS[1])
 else
     return 0
 end
 ```
这段Lua脚本在执行的时候要把的lockValue作为ARGV[1]的值传进去，把lockKey作为KEYS[1]的值传进去。现在来看看解锁的java代码

 ```
public void unlock() {
    // 使用lua脚本进行原子删除操作
    String checkAndDelScript = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                                "return redis.call('del', KEYS[1]) " +
                                "else " +
                                "return 0 " +
                                "end";
    jedis.eval(checkAndDelScript, 1, lockKey, lockValue);
}
 ```
 好了，解锁操作也确保了原子性了，就差过期时间待解决了。
 
 ##### 3.2.6 确保过期时间大于业务执行时间
 抽象类RedisLock增加一个boolean类型的属性isOpenExpirationRenewal，用来标识是否开启定时刷新过期时间。
 在增加一个scheduleExpirationRenewal方法用于开启刷新过期时间的线程。
 
 ```
 public abstract class RedisLock implements Lock {
 	//...
 
     protected volatile boolean isOpenExpirationRenewal = true;
 
     /**
      * 开启定时刷新
      */
     protected void scheduleExpirationRenewal(){
         Thread renewalThread = new Thread(new ExpirationRenewal());
         renewalThread.start();
     }
 
     /**
      * 刷新key的过期时间
      */
     private class ExpirationRenewal implements Runnable{
         @Override
         public void run() {
             while (isOpenExpirationRenewal){
                 System.out.println("执行延迟失效时间中...");
 
                 String checkAndExpireScript = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                         "return redis.call('expire',KEYS[1],ARGV[2]) " +
                         "else " +
                         "return 0 end";
                 jedis.eval(checkAndExpireScript, 1, lockKey, lockValue, "30");
 
                 //休眠10秒
                 sleepBySencond(10);
             }
         }
     }
 }
 ```
加锁代码在获取锁成功后将isOpenExpirationRenewal置为true，并且调用scheduleExpirationRenewal方法，开启刷新过期时间的线程。

 ```
 public void lock() {
     while (true) {
         String result = jedis.set(lockKey, lockValue, NOT_EXIST, SECONDS, 30);
         if (OK.equals(result)) {
             System.out.println("线程id:"+Thread.currentThread().getId() + "加锁成功!时间:"+LocalTime.now());
 
             //开启定时刷新过期时间
             isOpenExpirationRenewal = true;
             scheduleExpirationRenewal();
             break;
         }
         System.out.println("线程id:"+Thread.currentThread().getId() + "获取锁失败，休眠10秒!时间:"+LocalTime.now());
         //休眠10秒
         sleepBySencond(10);
     }
 }
 ```
解锁代码增加一行代码，将isOpenExpirationRenewal属性置为false，停止刷新过期时间的线程轮询。

 ```
 public void unlock() {
     //...
     isOpenExpirationRenewal = false;
 }
 ```
#### 3.4 测试类

 ```
 public void testLockCase5() {
     //定义线程池
     ThreadPoolExecutor pool = new ThreadPoolExecutor(0, 10,
                                                     1, TimeUnit.SECONDS,
                                                     new SynchronousQueue<>());
 
     //添加10个线程获取锁
     for (int i = 0; i < 10; i++) {
         pool.submit(() -> {
             try {
                 Jedis jedis = new Jedis("localhost");
                 LockCase5 lock = new LockCase5(jedis, lockName);
                 lock.lock();
 
                 //模拟业务执行15秒
                 lock.sleepBySencond(15);
 
                 lock.unlock();
             } catch (Exception e){
                 e.printStackTrace();
             }
         });
     }
 
     //当线程池中的线程数为0时，退出
     while (pool.getPoolSize() != 0) {}
 }
 ```
结果如下图

![](http://io.dbbaxbb.cn/assets/images/2018/docker/1653b1ac51325e07) <br/>
