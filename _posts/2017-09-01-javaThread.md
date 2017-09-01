---
layout: post
title: java Thread状态深入了解
keywords: java Thread 
desc: java thread 的各种状态切换
photoUrl:
---

# java Thread状态深入了解

## 线程

线程是进程中的一个单一顺序的执行单元，也被称为轻量进程（lightweight process）。线程有自己的必须资源，同时与其他线程进程共享全部资源。同一个进程中，线程是并发执行的。由于线程的之间的相互制约，线程有就绪、阻塞、运行、结束等状态。

## java中的线程

在java中运行一个线程很简单，通过继承Thread或者实现Runnable接口就可以创建一个线程。
```java
class MyThread exends Thread{
	
	@Override
	public void run(){

	}
}

class MyRunnable implements Runnable{

	@Override
	public void run(){

	}	
}
```
启动一个线程
```java
public static void main(String agrs[]){
	new MyThread().start();//继承启动
	new Thread(new MyRunnable()).start();//接口启动
}

```

## 验证java线程的并发性
在各自的run方法中 for循环输出 各自的名称
```java
	for(int i=0;i<100;i++){
		System.out.println("this is implements Runnable");
    }

    for(int i=0;i<100;i++){
		System.out.println("this is extends Thread");
    }
```

## 线程锁
下面方法执行后,数据少于1w
```java
public static void ThreadMuliteTest() throws InterruptedException {
        final List<String> list=new ArrayList();
        for(int i=0;i<10000;i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    list.add(Thread.currentThread().getName());
                }
            }).start();
        }
        console.info("等20s");
        Thread.sleep(20000);
        System.out.println(list.size());
}
```
线程如果同时执行就会产生一个问题 他们看到list里的数据是一样多的，插入的时候就会互相抵掉对方插入的数据，解决办法是排队，加锁
```java
public static void ThreadMuliteTest() throws InterruptedException {
        final List<String> list=new ArrayList();
        for(int i=0;i<10000;i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                	synchronized (list) {
                    	list.add(Thread.currentThread().getName());
                	}
                }
            }).start();
        }
        console.info("等20s");
        Thread.sleep(20000);
        System.out.println(list.size());
}

```

## 死锁

当线程互相等待对方释放自己需要的资源时 就会产生死锁

```java
public static void diedLock(){
        Object lock1=new Object();
        Object lock2=new Object();
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (lock1){
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    synchronized (lock2){
                        console.info("运行了！");
                    }
                }
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (lock2){
                    synchronized (lock1){
                        console.info("运行了！");
                    }
                }
            }
        }).start();
    }
```
## java线程的状态

![状态转换图](/images/20170901171737.png)

* NEW 新建状态 
   new 之后 start之前 都是属于NEW状态，一旦状态变成其他 则不会再回到NEW状态
* BLOCKED 阻塞状态
	如果获取不到锁 则进入阻塞状态 直到获取到锁之后
* RUNNABLE 执行状态
	在run方法里的时候 线程处于执行状态 一旦run方法结束 则状态不再是执行状态
* WAITING 等待状态
	wait调用后，睡眠不参与抢锁 等待notify唤醒
* TIMED_WAITING 等待超时状态
	当调用wait(long timeout) Thread.sleep方法时 线程处于等待状态 超时后并不返回 而是等待获取锁后返回
* TERMINATED 结束状态
    在run方法结束后线程处于结束状态

```java

public static void stateTest() throws InterruptedException {
        Thread t=new Thread(new Runnable() {
            @Override
            public void run() {
                long startTime=System.nanoTime();
                /**
                 * Thread.currentThread() 获取当前线程
                 * Thread.currentThread()===t true
                 * 打印当前线程状态
                 */
                printThreadState(Thread.currentThread());
                synchronized (lock){
                    try {
                        lock.wait();//放弃抢锁 处于WATING状态
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                console.info(System.nanoTime()-startTime+"纳秒");
            }
        });
        printThreadState(t);//查看new之后的thread状态
        synchronized (lock) {
            t.start();
            while (true) {
                printThreadState(t);//查看thread状态
                if (t.getState() == Thread.State.TERMINATED) {
                    break;
                } else if (t.getState() == Thread.State.TIMED_WAITING) {
                    synchronized (lock) {
                        Thread.sleep(20000);
                    }
                } else if (t.getState() == Thread.State.WAITING) {
                    synchronized (lock) {
                        lock.notify();//通知wait的线程 醒来抢锁
                    }
                } else if(t.getState()== Thread.State.BLOCKED){
                    lock.wait(100);//唤醒阻塞的线程
                    continue;
                }
                Thread.sleep(100);//休眠100ms
            }
        }
    }

```

## 唤醒线程

当线程处于TIMED_WAITING状态的时候 可以调用interrupt唤醒它
```java
public static void awakeThread() throws InterruptedException {
        Thread t=new Thread(new Runnable() {
            @Override
            public void run() {
                long startTime=System.currentTimeMillis();
                try {
                    synchronized (lock) {
                        lock.wait(20000);//Thread.sleep(20000) thread.join() lock.wait();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                console.info("唤醒成功！"+(System.currentTimeMillis()-startTime)+"s");
            }
        });
        t.start();
        Thread.sleep(100);
        t.interrupt();
    }
```