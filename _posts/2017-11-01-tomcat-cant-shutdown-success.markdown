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

>在执行shutdown.sh 发现tomcat 进程依然存在

##原因

有一些非deamon thread 存在，这种情况下，tomcat关闭后，整个进程并不会结束;

##解决方法


### 从程序上根本解决(要求较高，项目底层代码的修改） 

> 在项目中找到对应new Thread的地方setDaemon(true)，后面shutdown就没有tomcat进程了;

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








