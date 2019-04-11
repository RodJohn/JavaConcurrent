

# 实现

![](https://github.com/RodJohn/JavaConcurrent/blob/master/image/%E5%B9%B6%E5%8F%91%E9%94%81_ReentranLock.png)


	ReentrantLock委托内部的Sync实现了Lock接口
	Sync继承了AQS作为同步队列
	Sync的子类NonfairSync和FairSync分别提供了公平和非公平的获取方式
	
	ReentrantLock的State
	当 state = 0 时，表示锁未被占用。
	当 state > 0 时，表示锁已经占用。 数值表示线程的重入次数


	
	
	
# ReentrantLock


构造方法
 
	public ReentrantLock() {
		sync = new NonfairSync();
	}

	public ReentrantLock(boolean fair) {
		sync = fair ? new FairSync() : new NonfairSync();
	}

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

作用

	nonfairTryAcquire(int acquires) 方法，非公平锁的方式获得锁

代码

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

流程


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
	lock和tryAchive都是非公平的，直接使用CAS进行抢占锁
	

## lock

代码

	@Override
	final void lock() {
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
	}

流程


	通过CAS操作抢占锁
	若成功，则设置当前线程为排他线程
	若失败，执行AQS的独占式抢占锁
	先进行CAS抢占，已经能体现出非公平锁的特点。
	因为，此时有可能有 N + 1 个线程正在获得锁，其中 1 个线程已经获得到锁，释放的瞬间，恰好被新的线程抢夺到，而不是排队的 N 个线程。


## tryAcquire

	protected final boolean tryAcquire(int acquires) {
		return nonfairTryAcquire(acquires);
	}



# FairSync

	FairSync 是 ReentrantLock 的内部静态类，实现 Sync 抽象类，公平锁实现类。
	公平锁在获取同步状态时，会先判断即自己不是首个等待获取同步状态的节点

## lock
	
	final void lock() {
		acquire(1);
	}

## tryAcquire

	tryAcquire(int acquires) 实现方法，公平的方式，获得同步状态。
代码

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

作用

	判断自己不是首个等待获取同步状态的节点
	
	
代码

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

流程

	没人排队，或者头结点的下一个节点就是自己





