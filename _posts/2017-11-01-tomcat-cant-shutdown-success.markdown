---
layout:     post
title:      "tomcat7关闭(shutdown.sh)后进程依旧存在"
subtitle:   " \"原因:有一些非deamon thread 存在阻塞Tomcat\""
date:       2017-11-01
author:     "ThreeEggs"
header-img: "img/20171101180758.png"
catalog: true
tags:
    - Tomcat
    - 现象
---



##现象

```bash
[root@shj-124 ~]# sh /opt/web/tomcat.xsimple/bin/shutdown.sh 
Using CATALINA_BASE:   /opt/web/tomcat.xsimple
Using CATALINA_HOME:   /opt/web/tomcat.xsimple
Using CATALINA_TMPDIR: /opt/web/tomcat.xsimple/temp
Using JRE_HOME:        /usr
Using CLASSPATH:       /opt/web/tomcat.xsimple/bin/bootstrap.jar:/opt/web/tomcat.xsimple/bin/tomcat-juli.jar
[root@shj-124 ~]# ps -ef|grep tomcat
root      33369      1 88 14:00 pts/0    00:07:00 /usr/bin/java -Djava.util.logging.config.file=/opt/web/tomcat.xsimple/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -server -Dfile.encoding=UTF-8 -Xms10G -Xmx10G -Xss512k -XX:+AggressiveOpts -XX:+UseBiasedLocking -XX:PermSize=256M -XX:MaxPermSize=512M -XX:+DisableExplicitGC -XX:MaxTenuringThreshold=31 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -Djava.awt.headless=true -Djdk.tls.ephemeralDHKeySize=2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Djava.endorsed.dirs=/opt/web/tomcat.xsimple/endorsed -classpath /opt/web/tomcat.xsimple/bin/bootstrap.jar:/opt/web/tomcat.xsimple/bin/tomcat-juli.jar -Dcatalina.base=/opt/web/tomcat.xsimple -Dcatalina.home=/opt/web/tomcat.xsimple -Djava.io.tmpdir=/opt/web/tomcat.xsimple/temp org.apache.catalina.startup.Bootstrap start
root      33694  30517  0 14:08 pts/0    00:00:00 grep tomcat
[root@shj-124 ~]# kill -9 33369
[root@shj-124 ~]# 
```

>为什么在给Tomcat发出stop命令以后，Tomcat实例无法关闭？

##原因(两种)

###Tomcat的主线程没有结束（也即main函数没有执行结束）

第一种情况，如果发出stop命令以后，Tomcat主线程并没有结束，自然通过它启动的webapps也是无法关闭的。虽然可以确信Tomcat不可能有这个问题，但还是拉出Tomcat的关闭过程代码看看：

```java
public void start() {
    if (getServer() == null) {
        load();
    }
    if (getServer() == null) {
        log.fatal("Cannot start server. Server instance is not configured.");
        return;
    }
    long t1 = System.nanoTime();
    // Start the new server
    try {
        getServer().start();
    } catch (LifecycleException e) {
        log.fatal(sm.getString("catalina.serverStartFail"), e);
        try {
            getServer().destroy();
        } catch (LifecycleException e1) {
            log.debug("destroy() failed for failed Server ", e1);
        }
        return;
    }

    ……//省略

    if (await) {
        await();
        stop();
    }
}
```

首先需要清楚，Tomcat的正常关闭是通过socket发送命令的方式来触发的，Tomcat在启动完成以后会通过await()一直等待，知道接收到shutdown命令后退出，执行后面的stop()关闭Tomcat实例。

此后不会再有任何await，所以说Tomcat主线程是会正常关闭的。如果不相信，可以开启jpda debug一下，我就这么干了。

###Tomcat中启动的webapps有非daemon线程阻止了Tomcat进程的关闭

既然第一种可能不存在，那只能是webapps中有非daemon线程没有正常关闭了。为什么会这样？因为非Daemon线程被认为是工作线程，必须要主动关闭，而daemon线程属于后台线程，在非daemon线程关闭以后，daemon线程会自动关闭，典型的以main函数为入口的主线程便是非daemon的工作线程。

```java
public static void main(String[] args) {
    Thread thread = new Thread(() -> {
        while(true){
            LockSupport.parkNanos(1000 * 1000 * 3);
        }
    });
    thread.setDaemon(true);
    thread.start();

    try {
        Thread.sleep(1000 * 3);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

这里启动了一个非daemon线程，所以即使主线程执行完成以后，应用还是不会正常关闭，如果把线程改成daemon则会。

基于这个原理可以开始定位为什么Tomcat示例无法关闭了，可以通过jstack看看还有那些线程还在运行，之后逐一排除。



##解决方法


### 从程序上根本解决(要求较高，项目底层代码的修改） 

> 在项目中找到对应new Thread的地方setDaemon(true)，后面shutdown就没有tomcat进程了;

####如何通过jstack定位Thread

简单介绍一下可以使用jdk里面的jstack来找到出问题的Thread

* 找到tomcat当前运行的pid

  ```bash
  ps -ef |grep tomcat
  ```


* 找到jdk的目录，可以使用which java 命令,并切换到bin目录
* 在bin目录下执行jstack pid，查看每个thread，找出非deamon thread，进而判断这些thread为何没有停止;

```bash
root@shj-124 ~]#  ps -ef|grep tomcat
root      36965      1 99 17:20 pts/0    00:32:27 /usr/bin/java -Djava.util.logging.config.file=/opt/web/tomcat.xsimple/conf/logging.properties -Djava.util.logging.manag...
[root@shj-124 ~]# /usr/java/jdk1.7.0_79/bin/jstack 36965
2017-11-01 15:57:03
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.79-b02 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007fd210001000 nid=0x8855 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"DestroyJavaVM" prio=10 tid=0x00007fd2d866d000 nid=0x86d1 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Thread-24" daemon prio=10 tid=0x00007fd06800b000 nid=0x8806 waiting on condition [0x00007fd1756e2000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000059bd0e960> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:226)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2082)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1090)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:807)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"DubboSharedHandler-thread-1" daemon prio=10 tid=0x00007fd008006000 nid=0x8805 waiting on condition [0x00007fd21ca5f000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000005f74f54a0> (a java.util.concurrent.SynchronousQueue$TransferStack)
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:226)
	at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:460)
	at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:359)
	at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:942)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"pool-6-thread-13" prio=10 tid=0x00007fd05000b800 nid=0x87c3 waiting on condition [0x00007fd21d479000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000005d4b8a160> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at java.util.concurrent.ArrayBlockingQueue.take(ArrayBlockingQueue.java:374)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"pool-6-thread-12" prio=10 tid=0x00007fd050089800 nid=0x87c1 waiting on condition [0x00007fcfe37f2000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000005d4b8a160> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at java.util.concurrent.ArrayBlockingQueue.take(ArrayBlockingQueue.java:374)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"pool-6-thread-11" prio=10 tid=0x00007fd048094000 nid=0x87b9 waiting on condition [0x00007fcfe3b2f000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000005d4b8a160> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at java.util.concurrent.ArrayBlockingQueue.take(ArrayBlockingQueue.java:374)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
```

然后可以去catalina.out中搜索一下

> This is very likely to create a memory leak

![](https://raw.githubusercontent.com/wangmax0330/Myblog/master/image/20171101175206.png)

基本上如果在Tomcat启动的时候弹出这样的警告,会影响Tomcat关闭

然后一个一个Thread去改吧!!!

###从Tomcat上解决(kill进程)

- 解决方案一：

```bash
#查找到所有的tomcat进程
[root@shj-124 ~]#  ps -ef | grep tomcat
#然后逐一杀死它们
[root@shj-124 ~]#  ps -9 pid
```

- 解决方案二：

```bash
[root@shj-124 ~]#  kill -9 `ps -ef|grep tomcat|awk '{print $2}'`1212
[root@shj-124 ~]#  kill -9 `ps -ef|grep tomcat|awk '{print $2}'`
```

- 解决方案三：

  基本原理为启动tomcat时记录启动tomcat的进程id(pid),关闭时强制杀死该进程

第一步 ：vim修改tomcat下bin/catalina.sh文件，添加点东西，主要是记录tomcat的pid,如下:

```bash
#设置记录CATALINA_PID。
#该设置会在启动时候bin下新建一个CATALINA_PID文件
#关闭时候从CATALINA_PID文件找到pid，kill。。。同时删除CATALINA_PID文件
PRGDIR=`dirname "$PRG"`
if [ -z "$CATALINA_PID" ];then
 CATALINA_PID=$PRGDIR/CATALINA_PID
fi

##注意,你在哪个目录下运行startup.sh,CATALINA_PID就会在哪个目录下生成
```

第二步 vim tomcat的shutdown.sh文件,在最后一行加上-force:

```bash
exec "$PRGDIR"/"$EXECUTABLE" stop -force "$@"
```








