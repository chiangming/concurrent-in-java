# The underlying implementation of the concurrency mechanism

java代码编译后转换成字节码 -&gt; 字节码被类加载器加载到JVM -&gt; JVM执行字节码转换成汇编指令至CPU执行

## volatile

轻量级synchronized,保证共享变量的“可见性”（一个线程修改一个共享变量时，另一个线程能读到这个修改的值），它不会引起线程上下文的切换，因此执行成本比synchronized更低。有valatile变量修饰的共享变量进行写操作的时候会加Lock前缀指令：

1. 将当前处理器缓存行的数据写回到系统内存
2. 一个处理器的缓存回写到内存会导致其他处理器的缓存无效。（每个处理器通过嗅探在总线上传播的数据来检查自己的缓存的值是否过期）

volatile变量具备以下特性：

* 可见性：对于一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入
* 原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种符合操作不具备原子性。

## synchronized

实现同步的基础即java中每一个对象都可以作为锁。普通同步方法，锁是当前实例对象。静态同步方法，锁是当先类的Class对象。对于同步方法块，锁是Synchonized括号内置配的对象。

## synchronized在JVM中的实现

JVM基于进入和退出Monitor对象来实现代码块同步，monitorenter指令在编译后插入到同步代码块开始位置，而monitorexit则是插入到方法结束处和异常处。JVM保证monitorenter和monitorexit配对。任何对象都有一个monitor与之关联，当一个monitor被持有（线程执行到monitorenter指令）后，对象将处于锁定状态。

## 锁的升级与对比

锁一共4种状态：无锁状态、偏向锁状态、轻量级锁状态、重量锁状态。状态伴随竞争情况升级不会降级。

偏向锁：当线程a访问同步块并获取锁时，会在对象头和帧帆中的锁记录里存储锁偏向的线程a的ID，该线程在进去和退出同步块时不需要进行CAS操作加解锁，只需要测试对象头是否存储着线程a的偏向锁。（当测试失败且偏向锁的标识为1时尝试将对象头的偏向锁指向线程a）。![](/assets/import2-1.png)![](/assets/import2-2.png)![](/assets/import2-3.png)

## 原子操作

原子操作意为“不可被中断的一个或者一系列操作”

如果多个处理器同时对共享变量进行读改写操作（例如i++），那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的。例如

CPU1 ：（i=1） =&gt; \( i+1\) =&gt; \(i=2\)

CPU2 ：（i=1） =&gt;\( i+1\) =&gt; \(i=2\)

多个处理器同时从各自缓存中读取变量i,分别进行加1操作，然后分别写入系统的内存中。那么想要保证读改写共享变量的操作是原子的，就必须保证CPU1读改写共享变量的时候，其他CPU不能操作缓存了该共享变量内存地址的缓存。

## 处理器处理原子操作

* 使用总线锁保证原子性

**总线锁：**处理器提供一个LOCK\#信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，该处理器从而独占共享内存。

* 使用缓存锁保证原子性

**缓存锁：**处理器修改内部的内存地址，缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存喊的数据时，会使缓存行无效。

## JAVA处理原子操作

1. 加锁
2. 循环CAS \(compareAndSet\)

从JAVA1.5开始，JDK并发包提供了AtomicBoolean、AtomicInteger等类来支持原子操作。下例使用CAS实现线程安全的计数器。

```java
private AtomicInteger atomicI = new AtomicInteger(0);
```

```java
private void safeCount() {
    for (;;) {
        int i = atomicI.get();
        boolean suc = atomicI.compareAndSet(i, ++i);
        if (suc) {
            break;
        }
    }
}
```

循环CAS存在的问题: ABA问题（无法判断A修改成B修改成A是否修改）、循环开销大、只保证一个共享变量的原子操作。

对应的解决方案：AtomicStampedReference解决ABA问题，JVM支持处理器提供的pause指令，AtomicReference解决多个共享变量。

JVM内部实现锁的方式（除了偏向锁），其他锁的实现方式都用了循环CAS\(一个线程想进入同步块时使用循环CAS获取锁，退出时同理释放锁\)

