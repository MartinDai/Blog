---
title: 记一次锁使用不当导致Dubbo线程阻塞问题
date: 2019-11-17 21:28:00
categories:
- 问题排查
---

### 背景

线上环境一个后台项目，提供基于dubbo实现的事件分发服务，最近突然出现心跳超时。

### 问题分析

**检查内存是否溢出**
```
jstat -gcutil 8166 1000
```
意料之中，内存正常，因为内部有接入内存溢出告警，如果是内存溢出应该有收到通知，但是这次没有溢出通知。

**查看线程栈**
```
jstack -l 8166
```

发现有大量`DubboServerHandler`开头的线程阻塞在一个同样的地方，脱敏简化后信息如下：
```
"DubboServerHandler-192.168.160.42:9184-thread-200" #372 daemon prio=5 os_prio=0 tid=0x0000000002342000 nid=0x252b waiting on condition [0x00007f2ce8deb000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000008a51c930> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
    at com.▇▇▇▇▇▇.QueueMiddleComponent.save(QueueMiddleComponent.java:317)
	...
	at com.alibaba.dubbo.remoting.transport.dispatcher.ChannelEventRunnable.run(ChannelEventRunnable.java:81)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```
可以看到线程正在申请加锁，找到相关人员确认了这个不是业务代码而是我们自己内部封装的一个框架逻辑，于是找到对应的项目代码拉到本地。

<!--more-->

找到对应的执行代码段
```
final ReentrantLock writeLock = this.writeLock;
writeLock.lock();
WriteBatch wb = createWriteBatch();
try{
    //业务代码
}finally{
    try {
        wb.close();
        writeLock.unlock();
    } catch (IOException e) {
        writeLock.unlock();
        throw new IllegalStateException(e.getMessage());
    }
}
```
加锁的代码就是`writeLock.lock();`这一句，而当前对象的`writeLock`是一个私有字段，只有通过这里加锁和解锁，于是怀疑是其他线程获取到该锁以后执行阻塞了。

但是通过检查完整的线程栈，发现并没有其他线程阻塞在该方法获取到锁以后的情况。

**查看内存对象**
```
jmap -dump:format=b,file=8166.dump 8166
```
通过以上命令导出内存dump文件下载到本地，使用VisualVM加载dump文件。

找到`QueueMiddleComponent`对象，发现只有一个实例，其中`writeLock`字段的关键信息如下
![1](https://imgs.doodl6.com/problem/record-once-incorrect-use-lock/1.webp)

可以看到这个锁当前被名为`DubboServerHandler-192.168.160.42:9184-thread-93`的线程所持有，于是再从线程栈里面找到这个线程的当前栈信息

```
"DubboServerHandler-192.168.160.42:9184-thread-93" #259 daemon prio=5 os_prio=0 tid=0x0000000004b92800 nid=0x22b5 waiting on condition [0x00007f2cedc33000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000008a518f80> (a java.util.concurrent.SynchronousQueue$TransferStack)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:458)
	at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
	at java.util.concurrent.SynchronousQueue.take(SynchronousQueue.java:924)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```
可以看到这个线程已经执行完框架层的代码，正在等待新的请求，意味着这个锁已经不会再被释放了，那么是什么原因导致这个线程在执行完框架代码的时候没有释放`writeLock`这个锁的呢？

### 定位原因

相信眼尖的同学已经发现了上面那段加锁解锁代码中的问题了,解释如下

```
try {
        //如果wb.close()抛出异常,则不会执行unlock
        wb.close();
        writeLock.unlock();
    } catch (IOException e) {
        //如果抛出的不是IOException,则不会执行unlock
        writeLock.unlock();
        throw new IllegalStateException(e.getMessage());
    }
```
就是说在代码`wb.close()`执行的时候抛出了非`IOException`异常，然后没有被捕获住，所以导致释放锁代码`writeLock.unlock();`没有被执行就结束了。

### 解决问题

知道了原因，解决就简单了，只要稍微改造一下就可以了
```
try {
        wb.close();
    } catch (IOException e) {
        throw new IllegalStateException(e.getMessage());
    } finally {
        writeLock.unlock();
    }
```
只要把`writeLock.unlock();`代码移动到`finally`块中，保证即时出现异常，也能正常解锁就行了。


### 总结
简单来说，这是一次使用锁不恰当而导致的连锁反应，，因为其中一个线程异常退出没有解锁，导致其他进来的线程一旦进入到这个方法就会被阻塞，dubbo的线程数是有限的（默认200），当所有线程都被阻塞的时候，dubbo就完全不能提供服务了。

吸取一下经验

1. 解锁代码要放在`finally`块中，保证即使线程异常，也能正常解锁。
2. 如果需要加锁执行的代码，最好能做成异步执行，这样即使阻塞也只是阻塞异步线程池，不会影响主工作线程的正常执行。
