# **The challenges of concurrent programming**

并发编程所面临的主要挑战：

1. 上下文切换
2. 死锁
3. 软硬件资源限制

## 1.上下文切换

**上下文切换**：CPU通过时间片分配算法，切换时间片前需要保存上一个任务的状态。即使单核处理器也支持多线程执行代码。并发执行代码效率不一定比串行执行高，其原因在于线程有创建和上下文切换的开销（Lmbench3和vmstat可以测量上下文切换的时长和次数）。多线程在竞争锁时会引起上下文切换。

**减少上下文切换**：

1. 无锁并发编程：例如将数据ID按Hash算法取模分段，不同线程处理不同分段。
2. CAS算法：JAVA的Atomic包
3. 使用最小线程：避免大量线程等待
4. 协程：单线程实现多任务调度

jstack命令查看dump线程信息\(jps获取线程号\)

> jstack 1889 &gt; ./dumpmessage

统计线程状态查看

> grep java.lang.Thread.State dumpmessage \| awk '{print $2$3$4$5}' \|sort \| uniq -c

## 2. 死锁

在IDEA中可以使用![](/assets/import1-1.png)在调试过程中查看线程dump信息\(参考[IDEA调试文档](http://qinghua.github.io/intellij-idea-debug/)\)![](/assets/import1-2.png)

避免死锁：

1. 避免一个线程同时获取多个锁
2. 避免一个线程在锁内同时占用多个资源
3. 使用定时锁Lock.tryLock\(timeout\)替代内部锁
4. 数据库加解锁在一个连接中

# 3. 资源限制

硬件资源限制：使用Hadoop或者自己搭建集群，不同机器处理不同数据“数据ID%机器数”。

软件资源限制：使用资源池复用，例如将数据库和socket连接复用。

总结：强烈建议使用JDK并发包提供的并发容器和工具类来解决并发问题。

