


2. 共享式
共享式与独占式的最主要区别在于，同一时刻：

独占式只能有一个线程获取同步状态。
共享式可以有多个线程获取同步状态。
例如，读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。参见 ReentrantReadWriteLock 。



# acquireShared

public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}


调用 #tryAcquireShared(int arg) 方法，尝试获取同步状态，
获取成功则设置锁状态并返回大于等于 0 ，否则获取失败，返回小于 0 。若获取成功，直接返回，不用线程阻塞，自旋直到获得同步状态成功。

#tryAcquireShared(int arg) 方法，需要自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态

# doAcquireShared

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


设置新的首节点，并根据条件，唤醒下一个节点。
这里和独占式同步状态获取很大的不同：通过这样的方式，不断唤醒下一个共享式同步状态， 从而实现同步状态被多个线程的共享获取。


# setHeadAndPropagate

设置新的首节点，并根据条件，唤醒下一个节点。



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



# doReleaseShared

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
					continue;                // loop on failed CAS
			}
			if (h == head)                   // loop if head changed
				break;
		}
	}




