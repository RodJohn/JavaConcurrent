
# 锁

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时
访问共享资源（但是有些锁可以允许多个线程并发的访问共享资源，比如读写锁）。
在Lock接
口出现之前，Java程序是靠synchronized关键字实现锁功能的，而Java SE 5之后，并发包中新增
了Lock接口（以及相关实现类）用来实现锁功能

，它提供了与synchronized关键字类似的同步功
能，只是在使用时需要显式地获取和释放锁。虽然它缺少了（通过synchronized块或者方法所提
供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以
及超时获取锁等多种synchronized关键字所不具备的同步特性。

synchronized

	使用synchronized关键字将会隐式地获取锁，但是它将锁的获取和释放固化了，也就是先
	获取再释放。当然，这种方式简化了同步的管理，可是扩展性没有显示的锁获取和释放来的
	好


Lock接口

	Lock接口提供的synchronized关键字不具备的主要特性

	特性	描述
	尝试非阻塞的获取锁	当前线程尝试获取锁，如果如果这一时刻锁没有被其它的线程获取到，则成功获取并持有锁
	能被中断的获取锁	获取到锁的线程能够响应中断，当前获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放
	超时获取锁	在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回



# Lock

	Lock 接口
	java.util.concurrent.locks.Lock 接口，定义方法如下：

	void lock();
	void lockInterruptibly() throws InterruptedException;
	boolean tryLock();
	boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

	void unlock();

	Condition newCondition();

	Lock基本Api

	方法名称	描述
	void lock()	获取锁，调用该方法线程将会获取锁，当获取锁后，从该方法返回
	void lockInterruptibly() throws InterruptedException	可中断获取锁，和lock方法的不同之处在于该方法会响应中断，即在锁的获取中可以中断当前线程
	boolean tryLock()	尝试非阻塞的获取锁，调用该方法立即返回，如果能够获取在返回true，否则返回true
	boolean tryLock(long time，TimeUnit unit) hrows InterruptedException	
	超时获取锁，当前线程在以下3中情况下回返回

	1、当前线程在超时时间内获取了锁

	2、当前线程在超时时间内被中断

	3、超时时间结束，返回false

	void unlock()	释放锁
	Condition newCondition()

	获取等待通知组件，该组件和当前锁绑定，当前线程只有获取了锁，才能调用该组件的wait()方法。调用后，当前线程将释放锁





