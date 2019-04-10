

# Acquire

作用

	独占式获取锁
	
代码

	public final void acquire(int arg) {
	  if (!tryAcquire(arg) &&
		  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		  selfInterrupt();
	}

流程

	尝试获取锁（tryAcquire）
	如果获取失败，
	将当前线程加到同步队列尾部（addWaiter）
	并且自旋直到获得同步状态成功（acquireQueued）（当前线程的前驱节点是头结点时，会再次尝试获取锁）
	同时也会检测线程是否需要休眠
	



## tryAcquire

	tryAcquire(int arg) 
	需要自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态 

## addWaiter

	将当前线程加入到 CLH 同步队列尾部
	（AQS内部Node的方法）

## acquireQueued

	自旋直到获得同步状态成功
	如果发现当前线程的前驱节点是头结点，且同步状态成功

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

	靠前驱节点判断当前线程是否应该被阻塞
	
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

	如果 #shouldParkAfterFailedAcquire(Node pred, Node node) 方法返回 true ，
	则调用parkAndCheckInterrupt() 方法，阻塞当前线程

	private final boolean parkAndCheckInterrupt() {
	    LockSupport.park(this);
	    return Thread.interrupted();
	}


	然后，在线程被唤醒时，调用 Thread#interrupted() 方法，返回当前线程是否被打断，并清理打断状态。所以，实际上，线程被唤醒有两种情况：
	第一种，当前节点(线程)的前序节点释放同步状态时，唤醒了该线程。详细解析，见 「2 unparkSuccessor」 。
	第二种，当前线程被打断导致唤醒。









# release

作用

	释放独占式锁

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
	
流程

	释放锁
	唤醒同步队列的头结点

## tryRelease

	tryRelease(int arg) 方法，去尝试释放同步状态，释放成功则设置锁状态并返回 true


## unparkSuccessor

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






  
