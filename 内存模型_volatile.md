

# 优点

volatile是轻量级的synchronized，
它在多处理器开发中保证了共享变量的“可见性”。
比synchronized的使用和执行成本更低，因为它不会引起线程上下文的切换和调度。




# 特性

  破坏缓存
  限制重排序
  
  
  volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的
读。
如果一个字段被声明成volatile，Java线程内存
模型确保所有线程看到这个变量的值是一致的。


特性


  ·可见性。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写
  入。
  ·原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不
  具有原子性。


内存

   从内存语义的角度来说，volatile写和锁的释放有相同的内存语义；volatile读与锁的获取有相同的内存语义

  当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存
  当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

  线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息

  happenbefore 

# 实现原理

  禁止重排序
  从编译器重排序规则和处理器内存屏障插入策略


 
 # 增强
 
 
 
volatile

作用

如果一个字段被声明成volatile，
Java线程内存模型确保所有线程看到这个变量的值是一致的。



volatile是如何来保证可见性

有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码，
，Lock前缀的指令在多核处理器下会引发了两件事情[1]。
1）将当前处理器缓存行的数据写回到系统内存。
2）这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据

优化

volatile的使用优化
著名的Java并发编程大师Doug lea在JDK 7的并发包里新增一个队列集合类LinkedTransferQueue，
它在使用volatile变量时，用一种追加字节的方式来优化队列出队和入队的性
能。

