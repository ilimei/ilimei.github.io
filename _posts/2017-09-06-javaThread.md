---
layout: post
title: java Thread深入了解(四)
keywords: java Thread 
desc: 悲观锁和乐观锁
photoUrl:
---

# java Thread深入了解(四)

## 概念介绍
悲观锁是指假设并发更新冲突会发生，所以不管冲突是否真的发生，都会使用锁机制。
相对悲观锁而言，乐观锁机制采取了更加宽松的加锁机制

## 秒杀案例
下面是模拟秒杀的环境 每个返回操作前必须休眠100ms模拟数据库和其他的耗时操作
```java
/**
 * 秒杀服务
 * Created by 51422 on 2017/9/6.
 */
public class MiaoSha {

    public static int goods=10;

    /**
     * 返回订单号
     * @return -1 表示无货 否则货号
     * @throws InterruptedException
     */
    public int getGoods() throws InterruptedException {
        //todo 在这里完成获取货物代码
        Thread.sleep(100);//处理延时100ms
        return -1;
    }
}


/**
 * 测试案例
 */
public void testMiaoSha(){
    final MiaoSha miaoSha=new MiaoSha();
    long st=System.currentTimeMillis();
    ScheduledExecutorService service = Executors.newScheduledThreadPool(500);
    for(int i=0;i<50000;i++){
        service.execute(new Runnable() {
            public void run() {
                try {
                    int good=miaoSha.getGoods();
                    if(good>0){
                        System.out.println("获取到订单"+good);
                    };
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
    service.shutdown();
    try {
        service.awaitTermination(2, TimeUnit.DAYS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("耗时"+(System.currentTimeMillis()-st)+"ms");
}

```

## 悲观锁处理

```java
/**
 * 秒杀服务
 * Created by 51422 on 2017/9/6.
 */
public class MiaoSha {

    public static int goods=10;

    /**
     * 返回订单号
     * @return -1 表示无货 否则货号
     * @throws InterruptedException
     */
    public int getGoods() throws InterruptedException {
        //todo 在这里完成获取货物代码
        synchronized(this){
        	if(goods>0){
        		int no=goods;
        		goods--;
        		Thread.sleep(100);//处理延时100ms
        		return no;
        	}
        	Thread.sleep(100);//处理延时100ms
        	return -1;
        }
    }
}
```

## 乐观锁处理

```java
/**
 * 秒杀服务
 * Created by 51422 on 2017/9/6.
 */
public class MiaoSha {

    public static int goods=10;
    public static int version=0;

    /**
     * 返回订单号
     * @return -1 表示无货 否则货号
     * @throws InterruptedException
     */
    public int getGoods() throws InterruptedException {
        //todo 在这里完成获取货物代码
        int v=-1;
        if(goods>0&&(v=version)>0){
        	if(v==version){
        		synchronized(this){
					if(v==version){
						version++;
						int no=goods;
		        		goods--;
		        		Thread.sleep(100);//处理延时100ms
		        		return no;
					}
        		}
        	}
        }
        Thread.sleep(100);//处理延时100ms
        return -1;
    }
}
```

线程进入的时候获取version 同当前version对比 如果相同则参加锁竞争 竞争获得锁后再次对比version 防止version过期 如果version还是同当前版本号相同 则获取订单，这样会有效的同步线程，加快服务处理速度且保证货物不会被超发

## 信号量Semaphore
锁能保证同时只有一个线程使用资源 不能让指定个数的线程同时访问资源，这个时候就需要信号量,可以保证指定个数的线程访问资源，达到限流或者限速，比如同时只有2个客户端可以连接到服务器并使用。
```java
public static void SamephoreTest(){
    Semaphore samephore=new Semaphore(2);
    for(int i=0;i<3;i++){
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    samephore.acquire();
                    System.out.println("我访问了！");
                    Thread.sleep(5000);//休眠5s
                    samephore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```
acquire获取许可 是个阻塞的方法 
release释放许可
acquire和release必须成对出现 否则会产生资源死锁