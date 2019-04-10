
# 锁

	锁是用来控制多个线程访问共享资源的方式，

	JDK1.5之前，使用synchronized实现锁功能
	JDK1.5之后，使用Lock接口用来实现锁功能

synchronized

	synchronized会自动获取锁和释放锁

Lock接口

	可以显式地获取和释放锁

	尝试非阻塞的获取锁	
	当前线程尝试获取锁，如果如果这一时刻锁没有被其它的线程获取到，则成功获取并持有锁
	能被中断的获取锁	
	获取到锁的线程能够响应中断，当前获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放
	超时获取锁	
	在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回



# Lock-Api

	void lock()	
	获取锁，调用该方法线程将会获取锁，当获取锁后，从该方法返回
	
	8阻塞
	
	void lockInterruptibly() 
	可中断获取锁，和lock方法的不同之处在于该方法会响应中断，即在锁的获取中可以中断当前线程
	
	boolean tryLock()
	尝试非阻塞的获取锁，调用该方法立即返回，如果能够获取在返回true，否则返回true
	
	boolean tryLock(long time，TimeUnit unit) hrows InterruptedException	
	超时获取锁，当前线程在以下3中情况下回返回


	void unlock()	释放锁
	
	Condition newCondition()

	获取等待通知组件，该组件和当前锁绑定，当前线程只有获取了锁，才能调用该组件的wait()方法。调用后，当前线程将释放锁


# 使用方法




