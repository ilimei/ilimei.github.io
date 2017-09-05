---
layout: post
title: java Thread深入了解(三)
keywords: java Thread 
desc: 线程同步和线程池
photoUrl:
---

# java Thread深入了解(三)

## volatile
如下代码
```java
static byte value=0;
static boolean finish=false;

public static void testVolatile() throws InterruptedException {
    value=0;
    finish=false;
    new Thread(new Runnable() {
        @Override
        public void run() {
            while(value==0&&!finish);
            console.info("value is "+value+" finish is "+finish);
        }
    }).start();
    Thread.sleep(1000);
    new Thread(new Runnable() {
        @Override
        public void run() {
            value=10;
            finish=true;
            console.info("has set the finish");
        }
    }).start();
}
```
第一个线程的死循环 并不会因为第二个线程的设置变量结束，这是因为cpu缓存的问题
```java
volatile static byte value=0;
```
如果将value的改为volatile变量那么第一个线程就会如期结束。因为每次读取value并不去使用缓存

## concurrent的BlockingQueue
BlockingQueue是一个线程安全的队列接口 ，提供了 put take poll等方法
实现类有 ArrayBlockingQueue LinkedBlockingQueue PriorityBlockingQueue
```java
public static class Message implements Comparable<Message>{

    public int value=0;

    public Message(int value) {
        this.value = value;
    }

    @Override
    public int compareTo(Message o) {
        return value-o.value;
    }
}

public static void blockingQueueTest(){
    PriorityBlockingQueue <Message> blockingDeque=new PriorityBlockingQueue <Message>();
    new Thread(new Runnable() {
        @Override
        public void run() {
            for(int i=0;i<100;i++) {
                try {
                    console.info(blockingDeque.take().value);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    },"消费者").start();
    new Thread(new Runnable() {
        @Override
        public void run() {
            blockingDeque.put(new Message(100));
            blockingDeque.put(new Message(0));
            blockingDeque.put(new Message(0));
            blockingDeque.put(new Message(100));
        }
    },"生产者").start();
}
```
看到消费者拿到的消息是排序过的

## concurren包的线程池
Executors.newScheduledThreadPool 创建固定大小的线程池
Executors.newFixedThreadPool     创建固定大小的线程池
Executors.newCachedThreadPool    创建缓存线程池 从0 开始 每个线程存货一分钟
Executors.newSingleThreadExecutor 创建只有一个线程的线程池

实际上都是创建了一个ThreadPoolExecutor 控制了不同参数
```java
 /**
  *
  *  corePoolSize 线程池最小保持线程数 即是他们处于空闲状态
  *  maximumPoolSize 线程池最大线程数
  *  keepAliveTime 如果线程数超出最小保持 那么线程最大空闲存活实际
  *  unit 存活时间单位单位
  *  初始工作的缓冲区
  */
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
                              }
```

## ReentrantReadWriteLock
读写锁 把读和写分别变成锁 比synchronized更轻量控制更方便
读锁排斥写锁 不排斥读锁
写锁排斥其他锁
```java
public static void ReadWriteLock(){
    ReadWriteLock reentrantLock=new ReentrantReadWriteLock();
    Lock readLock=reentrantLock.readLock();
    Lock writeLock=reentrantLock.writeLock();
    for(int i=0;i<2;i++)
    new Thread(new Runnable() {
        @Override
        public void run() {
            readLock.lock();
            console.info("获取到了读锁");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();
    new Thread(new Runnable() {
        @Override
        public void run() {
            writeLock.lock();
            console.info("获取到了写锁");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();

}
```