

重入锁ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对
资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。

特点

回忆在同步器一节中的示例（Mutex），同时考虑如下场景：当一个线程调用Mutex的lock()
方法获取锁之后，如果再次调用lock()方法，则该线程将会被自己所阻塞，原因是Mutex在实现
tryAcquire(int acquires)方法时没有考虑占有锁的线程再次获取锁的场景，而在调用
tryAcquire(int acquires)方法时返回了false，导致该线程被阻塞。简单地说，Mutex是一个不支持
重进入的锁。而synchronized关键字隐式的支持重进入，比如一个synchronized修饰的递归方
法，在方法执行时，执行线程在获取了锁之后仍能连续多次地获得该锁，而不像Mutex由于获
取了锁，而在下一次获取锁时出现阻塞自己的情况



# 结构

## 结构

ReentrantLock ，实现 Lock 接口，重入锁。

构造方法
 
	public ReentrantLock() {
		sync = new NonfairSync();
	}

	public ReentrantLock(boolean fair) {
		sync = fair ? new FairSync() : new NonfairSync();
	}


## 原理

	ReentrantLock 的实现方法，基本是对 Sync 的调用。

lock

	@Override
	public void lock() {
		sync.lock();
	}

unlock

	@Override
	public void unlock() {
		sync.release(1);
	}


newCondition

	@Override
	public Condition newCondition() {
		return sync.newCondition();
	}



# Sync

	Sync抽象类
	Sync实现同步器(AbstractQueuedSynchronizer)。
	它使用 AQS 的 state 字段，来表示当前锁的持有数量，从而实现可重入的特性。

## lock

	abstract void lock();


## nonfairTryAcquire

	nonfairTryAcquire(int acquires) 方法，非公平锁的方式获得锁

	final boolean nonfairTryAcquire(int acquires) {
		//当前线程
		final Thread current = Thread.currentThread();
		//获取同步状态
		int c = getState();
		//state == 0,表示没有该锁处于空闲状态
		if (c == 0) {
			//获取锁成功，设置为当前线程所有
			if (compareAndSetState(0, acquires)) {
				setExclusiveOwnerThread(current);
				return true;
			}
		}
		//线程重入
		//判断锁持有的线程是否为当前线程
		else if (current == getExclusiveOwnerThread()) {
			int nextc = c + acquires;
			if (nextc < 0) // overflow
				throw new Error("Maximum lock count exceeded");
			setState(nextc);
			return true;
		}
		return false;
	}

## tryRelease


	protected final boolean tryRelease(int releases) {
		// 减掉releases
		int c = getState() - releases;
		// 如果释放的不是持有锁的线程，抛出异常
		if (Thread.currentThread() != getExclusiveOwnerThread())
			throw new IllegalMonitorStateException();
		boolean free = false;
		// state == 0 表示已经释放完全了，其他线程可以获取同步状态了
		if (c == 0) {
			free = true;
			setExclusiveOwnerThread(null);
		}
		setState(c);
		return free;
	}


# NonfairSync

	NonfairSync实现 Sync 抽象类，非公平锁实现类。

## lock

	#lock() 实现方法，首先基于 AQS state 进行 CAS 操作，将 0 => 1 。
	若成功，则获取锁成功。若失败，执行 AQS 的正常的同步状态获取逻辑。
	优先基于 AQS state 进行 CAS 操作，已经能体现出非公平锁的特点。
	因为，此时有可能有 N + 1 个线程正在获得锁，其中 1 个线程已经获得到锁，释放的瞬间，恰好被新的线程抢夺到，而不是排队的 N 个线程。



	@Override
	final void lock() {
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
	}

## tryAcquire

	tryAcquire(int acquires) 实现方法，非公平的方式，获得同步状态。代码如下：
	直接调用 #nonfairTryAcquire(int acquires) 方法，非公平锁的方式获得锁。

	protected final boolean tryAcquire(int acquires) {
		return nonfairTryAcquire(acquires);
	}



# FairSync

FairSync 是 ReentrantLock 的内部静态类，实现 Sync 抽象类，公平锁实现类。

## lock
	
	#lock() 实现方法
	直接执行 AQS 的正常的同步状态获取逻辑。

	final void lock() {
		acquire(1);
	}

## tryAcquire

	tryAcquire(int acquires) 实现方法，公平的方式，获得同步状态。


	protected final boolean tryAcquire(int acquires) {
		final Thread current = Thread.currentThread();
		int c = getState();
		if (c == 0) {
			if (!hasQueuedPredecessors() && 
					compareAndSetState(0, acquires)) {
				setExclusiveOwnerThread(current);
				return true;
			}
		}
		else if (current == getExclusiveOwnerThread()) {
			int nextc = c + acquires;
			if (nextc < 0)
				throw new Error("Maximum lock count exceeded");
			setState(nextc);
			return true;
		}
		return false;
	}


## hasQueuedPredecessors

	比较非公平锁和公平锁获取同步状态的过程，会发现两者唯一的区别就在于，
	公平锁在获取同步状态时多了一个限制条件 <1> 处的 
	#hasQueuedPredecessors() 方法，是否有前序节点，即自己不是首个等待获取同步状态的节点。代码如下：
	该方法主要做一件事情：主要是判断当前线程是否位于 CLH 同步队列中的第一个。如果是则返回 true ，否则返回 false 。

	// AbstractQueuedSynchronizer.java
	public final boolean hasQueuedPredecessors() {
		Node t = tail;  //尾节点
		Node h = head;  //头节点
		Node s;

		//头节点 != 尾节点
		//同步队列第一个节点不为null
		//当前线程是同步队列第一个节点
		return h != t &&
				((s = h.next) == null || s.thread != Thread.currentThread());
	}


# ReentrantLock 与 synchronized 的区别
前面提到 ReentrantLock 提供了比 synchronized 更加灵活和强大的锁机制，那么它的灵活和强大之处在哪里呢？他们之间又有什么相异之处呢？

首先他们肯定具有相同的功能和内存语义。

与 synchronized 相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
ReentrantLock 还提供了条件 Condition ，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock 更加适合（以后会阐述Condition）。
ReentrantLock 提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而 synchronized 则一旦进入锁请求要么成功要么阻塞，所以相比 synchronized 而言，ReentrantLock会不容易产生死锁些。
ReentrantLock 支持更加灵活的同步代码块，但是使用 synchronized 时，只能在同一个 synchronized 块结构中获取和释放。注意，ReentrantLock 的锁释放一定要在 finally 中处理，否则可能会产生严重的后果。
ReentrantLock 支持中断处理，且性能较 synchronized 会好些。





