

CAS ，Compare And Swap ，即比较并交换。
Doug Lea 大神在实现同步组件时，大量使用CAS 技术，鬼斧神工地实现了Java 多线程的并发操作。
整个 AQS 同步组件、Atomic 原子类操作等等都是基 CAS 实现的，甚至 ConcurrentHashMap 在 JDK 1.8 的版本中，也调整为 CAS + synchronized 。
可以说，CAS 是整个 J.U.C 的基石。

# 作用

CAS 虽然高效地解决了原子操作，

# CPU 原子操作

CAS 可以保证一次的读-改-写操作是原子操作，在单处理器上该操作容易实现，但是在多处理器上实现就有点儿复杂了。
CPU 提供了两种方法来实现多处理器的原子操作：总线加锁或者缓存加锁。

总线加锁

总线加锁就是就是使用处理器提供的一个 LOCK# 信号，
当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占使用共享内存。
会把CPU和内存之间的通信锁住了，在锁定期间，其他处理器都不能其他内存地址的数据，其开销有点儿大。

缓存加锁

缓存加锁，就是缓存在内存区域的数据
如果在加锁期间，当它执行锁操作写回内存时，处理器不再输 出LOCK# 信号，而是修改内部的内存地址，利用缓存一致性协议来保证原子性。
缓存一致性机制可以保证同一个内存区域的数据仅能被一个处理器修改，
也就是说当 CPU1 修改缓存行中的 i 时使用缓存锁定，那么 CPU2 就不能同时缓存了 i 的缓存行。

# 缺陷

但是还是存在一些缺陷的，


循环时间太长

	如果自旋 CAS 长时间地不成功，则会给 CPU 带来非常大的开销。
	在 J.U.C 中，有些地方就限制了 CAS 自旋的次数，例如： BlockingQueue 的 SynchronousQueue 。

只能保证一个共享变量原子操作

	看了 CAS 的实现就知道这只能针对一个共享变量，
	如果是多个共享变量就只能使用锁了，当然如果你有办法把多个变量整成一个变量，利用 CAS 也不错。
	例如读写锁中 state 的高低位。

ABA问题

	CAS 需要检查操作值有没有发生改变，如果没有发生改变则更新。
	但是存在这样一种情况：如果一个值原来是 A，变成了 B，然后又变成了 A，那么在 CAS 检查的时候会发现没有改变，但是实质上它已经发生了改变，这就是所谓的ABA问题。
	对于 ABA 问题其解决方案是加上版本号，即在每个变量都加上一个版本号，每次改变时加 1 ，即 A —> B —> A ，变成1A —> 2B —> 3A 。

# ABA 

 解决方案 AtomicStampedReference
CAS 的 ABA 隐患问题，解决方案则是版本号，Java 提供了 AtomicStampedReference 来解决。AtomicStampedReference 通过包装 [E,Integer] 的元组，来对对象标记版本戳 stamp ，从而避免 ABA 问题。对于上面的案例，应该线程 1 会失败。






