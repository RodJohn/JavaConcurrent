
# AcquireShared

流程


	尝试获取共享锁
	获取失败，自旋直到获得同步状态成功。
	
	当大于0的时候，就证明我们现在共享锁的资源充足，可能目前有线程阻塞在队列中，我们需要去唤醒当前节点的下一个节点，这就是共享锁唤醒的传播性。


代码

	public final void acquireShared(int arg) {
	    if (tryAcquireShared(arg) < 0)
		doAcquireShared(arg);
	}



## tryAcquireShared


	tryAcquireShared是共享方式尝试获取资源。
	负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源
	

## doAcquireShared

流程

	将线程加入同步队列尾部，并自旋
	如果前驱节点是头结点，则尝试获取同步锁
	-设置新的首节点，并根据条件，唤醒下一个节点。
	这里和独占式同步状态获取很大的不同：通过这样的方式，不断唤醒下一个共享式同步状态， 从而实现同步状态被多个线程的共享获取。


代码

	private void doAcquireShared(int arg) {
		// 共享式节点
		final Node node = addWaiter(Node.SHARED);
		boolean failed = true;
		try {
			boolean interrupted = false;
			for (;;) {
				// 前驱节点
				final Node p = node.predecessor();
				// 如果其前驱节点，获取同步状态
				if (p == head) {
					// 尝试获取同步
					int r = tryAcquireShared(arg);
					if (r >= 0) {
						setHeadAndPropagate(node, r);
						p.next = null; // help GC
						if (interrupted)
							selfInterrupt();
						failed = false;
						return;
					}
				}
				if (shouldParkAfterFailedAcquire(p, node) &&
						parkAndCheckInterrupt())
					interrupted = true;
			}
		} finally {
			if (failed)
				cancelAcquire(node);
		}
	}

### addWaiter

### setHeadAndPropagate

作用

	设置新的首节点，并根据条件，唤醒下一个节点。
	当大于0的时候，就证明我们现在共享锁的资源充足，
	可能目前有线程阻塞在队列中，我们需要去唤醒当前节点的下一个节点，这就是共享锁唤醒的传播性。


代码

	private void setHeadAndPropagate(Node node, int propagate) {
		Node h = head; // Record old head for check below
		setHead(node);
		/*
		 * Try to signal next queued node if:
		 *   Propagation was indicated by caller,
		 *     or was recorded (as h.waitStatus either before
		 *     or after setHead) by a previous operation
		 *     (note: this uses sign-check of waitStatus because
		 *      PROPAGATE status may transition to SIGNAL.)
		 * and
		 *   The next node is waiting in shared mode,
		 *     or we don't know, because it appears null
		 *
		 * The conservatism in both of these checks may cause
		 * unnecessary wake-ups, but only when there are multiple
		 * racing acquires/releases, so most need signals now or soon
		 * anyway.
		 */
		if (propagate > 0 || h == null || h.waitStatus < 0 ||
			(h = head) == null || h.waitStatus < 0) {
			Node s = node.next;
			if (s == null || s.isShared())
				doReleaseShared();
		}
	}



# releaseShared

代码

	public final boolean releaseShared(int arg) {
	    if (tryReleaseShared(arg)) {
	        doReleaseShared();
	        return true;
	    }
	    return false;
	}

## doReleaseShared

作用

	唤醒后续的共享式获取同步状态的节点


代码

	private void doReleaseShared() {
		/*
		 * Ensure that a release propagates, even if there are other
		 * in-progress acquires/releases.  This proceeds in the usual
		 * way of trying to unparkSuccessor of head if it needs
		 * signal. But if it does not, status is set to PROPAGATE to
		 * ensure that upon release, propagation continues.
		 * Additionally, we must loop in case a new node is added
		 * while we are doing this. Also, unlike other uses of
		 * unparkSuccessor, we need to know if CAS to reset status
		 * fails, if so rechecking.
		 */
		for (;;) {
			Node h = head;
			if (h != null && h != tail) {
				int ws = h.waitStatus;
				if (ws == Node.SIGNAL) {
					if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
						continue;            // loop to recheck cases
					unparkSuccessor(h);
				}
				else if (ws == 0 &&
						 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
					continue;           
			}
			if (h == head)                  
				break;
		}
	}




