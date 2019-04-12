

# Acquire

作用

	提供独占锁的获取流程
流程

	1.尝试获取锁（tryAcquire），如果成功则直接返回；
	2.如果获取失败，并标记为独占模式；
	3.将当前线程加到同步队列尾部（addWaiter）
	3.线程在等待队列中阻塞，被唤醒时会去尝试获取资源。获取到资源后才返回。。
	5. 如果线程在等待过程中被中断唤醒，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上

		
代码

	public final void acquire(int arg) {
	  if (!tryAcquire(arg) &&
		  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		  selfInterrupt();
	}





## tryAcquire

	子类提供的尝试获取锁的方法

## addWaiter

	通过CAS修改对象字段的方式，将当前线程加入到同步队列尾部
	（AQS内部Node的方法）

## acquireQueued

作用

	自旋直到获得同步状态成功
	如果发现当前线程的前驱节点是头结点，且同步状态成功


代码

	final boolean acquireQueued(final Node node, int arg) {
		// 记录是否获取同步状态成功
		boolean failed = true;
		try {
			// 记录过程中，是否发生线程中断
			boolean interrupted = false;
			/*
			 * 自旋过程，其实就是一个死循环而已
			 */
			for (;;) {
				// 当前线程的前驱节点
				final Node p = node.predecessor();
				// 当前线程的前驱节点是头结点，且同步状态成功
				if (p == head && tryAcquire(arg)) {
					setHead(node);
					p.next = null; // help GC
					failed = false;
					return interrupted;
				}
				// 获取失败，线程等待--具体后面介绍
				if (shouldParkAfterFailedAcquire(p, node) &&
						parkAndCheckInterrupt())
					interrupted = true;
			}
		} finally {
			// 获取同步状态发生异常，取消获取。
			if (failed)
				cancelAcquire(node);
		}
	}
	

### shouldParkAfterFailedAcquire

作用

	靠前驱节点判断当前线程是否应该被阻塞

流程

	如果当前节点的前驱节点的waitStatus是SIGNAL，返回true，表示当前节点应当park
	如果当前节点的前驱节点的waitStatus>0，相当于CANCELLED（因为状态值里面只有CANCELLED是大于0的），
	那么CANCELLED的节点作废，当前节点不断向前找并重新连接为双向队列，
	直到找到一个前驱节点waitStats不是CANCELLED的并且最靠近head节点的那一个为止。
	它的前驱节点不是SIGNAL状态且waitStatus<=0，利用CAS机制把前驱节点的waitStatus更新为SIGNAL状态。
	在这种情况下parkAndCheckInterrupt返回的是false，它还需要最后一次tryAcquire


代码

	private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
		// 获得前一个节点的等待状态
		int ws = pred.waitStatus;
		if (ws == Node.SIGNAL) //  Node.SIGNAL
			/*
			 * This node has already set status asking a release
			 * to signal it, so it can safely park.
			 */
			return true;
		if (ws > 0) { // Node.CANCEL
			/*
			 * Predecessor was cancelled. Skip over predecessors and
			 * indicate retry.
			 */
			do {
				node.prev = pred = pred.prev;
			} while (pred.waitStatus > 0);
			pred.next = node;
		} else { // 0 或者 Node.PROPAGATE
			/*
			 * waitStatus must be 0 or PROPAGATE.  Indicate that we
			 * need a signal, but don't park yet.  Caller will need to
			 * retry to make sure it cannot acquire before parking.
			 */
			compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
		}
		return false;
	}



### parkAndCheckInterrupt

流程

	阻塞当前线程
	如果这个线程被中断唤醒了
	Thread#interrupted()会清除当前线程的中断标记位并返回true，则使用interrupted记录中断发生
	如果是前序节点释放同步状态时唤醒了该线程
	/则进入循环尝试获取锁

代码

	private final boolean parkAndCheckInterrupt() {
	    LockSupport.park(this);
	    return Thread.interrupted();
	}


# acquireInterruptibly

作用

	响应中断的获取方式
	流程和acquire差不多
	在阻塞时如果被中断，不再是使用 interrupted 标志，而是直接抛出 InterruptedException 异常


代码

	public final void acquireInterruptibly(int arg) throws InterruptedException {
	    if (Thread.interrupted())
		throw new InterruptedException();
	    if (!tryAcquire(arg))
		doAcquireInterruptibly(arg);
	}

	private void doAcquireInterruptibly(int arg)
	    throws InterruptedException {
	    final Node node = addWaiter(Node.EXCLUSIVE);
	    boolean failed = true;
	    try {
		for (;;) {
		    final Node p = node.predecessor();
		    if (p == head && tryAcquire(arg)) {
			setHead(node);
			p.next = null; // help GC
			failed = false;
			return;
		    }
		    if (shouldParkAfterFailedAcquire(p, node) &&
			parkAndCheckInterrupt())
			throw new InterruptedException(); // <1>
		}
	    } finally {
		if (failed)
		    cancelAcquire(node);
	    }
	}





# release

作用

	释放锁后
	唤醒等待队列中最前边的那个未放弃的线程
	
代码

	public final boolean release(int arg) {
		if (tryRelease(arg)) {
			Node h = head;
			if (h != null && h.waitStatus != 0)
				unparkSuccessor(h);
			return true;
		}
		return false;
	}


## tryRelease

	tryRelease(int arg) 方法，去尝试释放同步状态，释放成功则设置锁状态并返回 true


## unparkSuccessor

流程

	

代码

	private void unparkSuccessor(Node node) {
	    //当前节点状态
	    int ws = node.waitStatus;
	    //当前状态 < 0 则设置为 0
	    if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);

	    //当前节点的后继节点
	    Node s = node.next;
	    //后继节点为null或者其状态 > 0 (超时或者被中断了)
	    if (s == null || s.waitStatus > 0) {
		s = null;
		//从tail节点来找可用节点
		for (Node t = tail; t != null && t != node; t = t.prev)
		    if (t.waitStatus <= 0)
			s = t;
	    }
	    //唤醒后继节点
	    if (s != null)
		LockSupport.unpark(s.thread);
	}






  
