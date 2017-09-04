---
layout: post
title: java Thread状态深入了解(二)
keywords: java Thread 
desc: 线程同步和线程池
photoUrl:
---

# java Thread状态深入了解(二)

## synchronized方法和synchronized代码段
synchronized方法 只有一个线程能访问，只有这个方法执行完之后其他线程才能访问
synchronized方法的锁是当前对象，也就是如果new多次是不能阻止多个线程访问代码的
```java
public class SyncTest {
	
	/**
	 * 对于SyncTest的对象 只有一个线程能访问
	 * 对于不同的SyncTest的对象可以有多个线程访问
	 */
	public  synchronized void say(){
        System.out.println("hello get the lock!");
        //休眠200s 不释放锁
        try {
            Thread.sleep(200 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

	/**
	 * 与synchronized方法相同
	 */
    public  void say(){
        synchronized(this) {
            System.out.println("hello get the lock!");
            //休眠200s 不释放锁
            try {
                Thread.sleep(200 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

	/**
	 * 同一时间只有一个线程能访问
	 * 与synchronized方法不同
	**/
    public  void say(){
        synchronized(SyncTest.class) {
            System.out.println("hello get the lock!");
            //休眠200s 不释放锁
            try {
                Thread.sleep(200 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
最为线程安全类 最常见的做法就是加synchronized方法

StringBuilder 
StringBuffer 
HashTable 
HashMap

## Thread类常用方法

名称 new Thread(String name) new Thread(Runnable run,String name)

获取当前Thread Thread.currentThread();

获取当前所有Thread的数量 Thread.activeCount();

## 线程变量ThreadLocal
ThreadLocal是一个特殊的模板变量 每个线程获取到的都是自己的值
```java
public class ThreadLocalTest{

    private static ThreadLocal<ThreadLocalTest> threadLocal=new ThreadLocal<>();
    private static ThreadLocalTest mainThreadLocal;

    /**
     * 创建当前线程的ThreadLocal对象
     */
    public static void prepare(){
        if(threadLocal.get()==null){
            threadLocal.set(new ThreadLocalTest());
        }
    }

	/**
	 * 主线程中调用
	 */
    public static void prepareMain(){
        if(mainThreadLocal!=null){
            throw new RuntimeException("只有一个线程可以调用prepareMain");
        }
        mainThreadLocal=new ThreadLocalTest();
        if(threadLocal.get()==null){
            threadLocal.set(mainThreadLocal);
        }
    }

    public static ThreadLocalTest getMain(){
        return mainThreadLocal;
    }

    public static ThreadLocalTest myThreadTest(){
        return threadLocal.get();
    }

    private ThreadLocalTest(){

    }

}

```
其实现原理 利用的是Thread.currentThread();

## concurrent包的CountDownLatch
一个辅助类，初始化的时候指定数据个数，没调用一次countDown()数字减少一个
await方法会阻塞 直到CountDownLatch里的数字为0
```java
CountDownLatch countDownLatch=new CountDownLatch(100);
for(int i=0;i<100;i++){
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(200);
                System.out.println(Thread.currentThread().getName()+" done!");
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();
}
try {
    countDownLatch.await();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.println("所有线程完成了!");
```
线程A , B ,C。 A,B运行完后运行C
```java
CountDownLatch countDownLatch=new CountDownLatch(2);
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+" done !");
        countDownLatch.countDown();
    }
},"A").start();
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+" done !");
        countDownLatch.countDown();
    }
},"B").start();
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+" done !");
    }
},"C").start();
```

## 线程池初步
```java
long startTime=System.currentTimeMillis();
CountDownLatch countDownLatch=new CountDownLatch(65535);
for(int i=0;i<65535;i++){
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(200);
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();
}
try {
    countDownLatch.await();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.println("所有线程完成了!"+(System.currentTimeMillis()-startTime)+"ms");
```
上面的代码预计消耗时间是200ms但是 实际远远超出了我们的预期，这是因为新建线程消耗了一部分时间，cpu拥挤，并不会有那么多线程同时运行，会排队一会。
通过线程池可以优化上述代码的速度
```java
/*
 * 空闲线程
*/
public class IdleThread extends Thread {

    public LinkedBlockingQueue<Runnable> queue;
    public boolean started = true;

    public IdleThread(LinkedBlockingQueue<Runnable> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        while (started) {
            try {
                Runnable runnable = queue.poll(100, TimeUnit.MILLISECONDS);
                if(runnable!=null)
                    runnable.run();
            } catch (InterruptedException e) {
            }
        }
    }

    public void kill() {
        started = false;
    }
}

/**
 * 基础线程池
 */
public class ThreadPool {

    /**
     * 池最小大小
     */
    private int minSize;

    /**
     * 池最大大小
     */
    private int maxSize;


    LinkedBlockingQueue<Runnable> tasks;

    List<IdleThread> idleThreads;

    public ThreadPool(int size){
        this.maxSize=size;
        this.minSize=size;
        init();
    }

    private void init(){
        tasks=new LinkedBlockingQueue<>(minSize);
        idleThreads=new ArrayList<>(minSize);
        for(int i=0;i<minSize;i++){
            IdleThread thread=new IdleThread(tasks);
            idleThreads.add(thread);
            thread.start();
        }
    }

    public void execute(Runnable runnable) throws InterruptedException {
        tasks.put(runnable);
    }

    public void shutdown(){
        try {
            for (IdleThread idleThread : idleThreads) {
                idleThread.kill();
                idleThread.interrupt();
            }
        }catch (Exception e){
            e.printStackTrace();;
        }
        tasks.clear();
    }

    public void waitForAll(){
        while(tasks.size()>0){
            try {
                Thread.sleep(0,1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
通过上述两类 重写一下上面的代码
```java
long startTime=System.currentTimeMillis();
CountDownLatch countDownLatch=new CountDownLatch(65535);
ThreadPool threadPool=new ThreadPool(9000);
for(int i=0;i<65535;i++){
    threadPool.execute(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(200);
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
}
try {
    countDownLatch.await();
    threadPool.shutdown();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.println("所有线程完成了!"+(System.currentTimeMillis()-startTime)+"ms");
```
可以看到速度比那个优化很多