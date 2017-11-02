---
layout:     post
title:      "Tomcat假死,无法响应前端请求原因分析"
subtitle:   ""
date:       2017-11-02
author:     "ThreeEggs"
header-img: "img/20171014171937.jpg"
catalog: true
tags:
    - Tomcat
    - 解决方法
---

#使用后台进程和Shutdown Hook友好地关闭Tomcat



严重的问题就是在JVM关闭时，行为不良的线程不会被关闭。

你可能会问：为什么这会成为问题……好吧，对程序员来说这真的算不上一个问题，只要随便写点代码就可以解决。但是对使用软件的人而言这却会带来不必要的麻烦。原因是这样会产生很多行为不良的线程，而执行Tomcat的shutdown.sh命令收效甚微。这时你不得不执行下面命令野蛮的杀掉web服务器：

```bash
ps -ef | grep java
```

先得到进程pid，然后

```bash
kill -9 <<pid>>
```

……接着需要有一大片web服务器需要重启，这种混乱绝对让人头痛。最后你执行shutdown.sh停止Tomcat。

在我最近的几篇博客里，我编写的那些行为不良的线程在run()方法开头都包含了下面的代码：

```java
Override
public void run() { 
  while (true) { 
    try { 
    
      DeferredResult<Message> result = resultQueue.take(); 
      Message message = queue.take(); 
 
      result.setResult(message); 
 
    } catch (InterruptedException e) { 
      throw new UpdateException("Cannot get latest update. " + e.getMessage(), e); 
    } 
  } 
}
```

在上面的代码里，我用了一个无限循环while(true)，这意味着线程会一直运行并且不会终止。

```java
@Override
public void run() { 
 
  sleep(5); // Sleep等待app重新加载 
 
  logger.info("The match has now started..."); 
  long now = System.currentTimeMillis(); 
  List<Message> matchUpdates = match.getUpdates(); 
 
  for (Message message : matchUpdates) { 
 
    delayUntilNextUpdate(now, message.getTime()); 
    logger.info("Add message to queue: {}", message.getMessageText()); 
    queue.add(message); 
  } 
  start = true; // 结束，重启 
  logger.warn("GAME OVER"); 
}
```

上面第二个示例中线程的行为同样很糟糕。线程会从MatchUpdates列表中取消息并在合适的时候添加到消息队列。唯一的可取之处是他们会抛出异常InterruptedException，如果正确处理线程可以正常停止。然而，没有人能确保这一点。



上面代码的一个有效地快速修正……只要确保创建所有线程都是后台线程。后台线程的定义是：在程序结束时，即使线程还在运行但不会阻止JVM退出。一个后台线程的例子就是JVM的垃圾回收线程。将线程设置为后台线程只需要调用：

```java
thread.setDaemon(true);
```

……接着执行shutdown.sh，然后砰的一声所有的线程都消失了。然而这种做法有一个问题：如果你的后台线程正在执行重要的任务，刚刚开始执行就被突然结束掉会导致丢失很多重要的数据该怎么办？



你需要确保所有线程都被友好地关闭，在关闭前完成所有正在执行的任务。本文接下来将为这些错误的线程给出一个修复，使用ShutdownHook让他们在关闭前互相协调。根据[文档](http://www.importnew.com/6255.html#addShutdownHook(java.lang.Thread))的描述：“一个shutdown hook就是一个初始化但没有启动的线程。 当虚拟机开始执行关闭程序时，它会启动所有已注册的shutdown hook（不按先后顺序）并且并发执行。”读完最后一句话，你可能已经猜到了你需要的就是创建一个负责关闭多有其他线程的线程并通过shutdown hook传递给JVM。只要在你已有线程的run() 方法里用几个小的class做一些手脚。



需要创建ShutdownService和Hook两个类。首先展示的是Hook类，它会将ShutdownService 连接到你的线程，代码如下：

```java
public class Hook { 
 
  private static final Logger logger = LoggerFactory.getLogger(Hook.class); 
 
  private boolean keepRunning = true; 
 
  private final Thread thread; 
 
  Hook(Thread thread) { 
    this.thread = thread; 
  } 
 
  /** 
   * @return True 如果后台线程继续运行
   */ 
  public boolean keepRunning() { 
    return keepRunning; 
  } 
 
  /** 
   * 告诉客户端后台线程关闭并等待友好地关闭
   */ 
  public void shutdown() { 
    keepRunning = false; 
    thread.interrupt(); 
    try { 
      thread.join(); 
    } catch (InterruptedException e) { 
      logger.error("Error shutting down thread with hook", e); 
    } 
  } 
}
```

