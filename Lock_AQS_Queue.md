


当前线程（Node）进入同步队列后，
就会进入一个自旋的过程，每个节点都会自省地观察，
当条件满足，获取到同步状态后，就可以从这个自旋过程中退出，否则会一直执行下去。

当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程
当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。


# Node

	static final class Node {

		// 共享
		static final Node SHARED = new Node();
		// 独占
		static final Node EXCLUSIVE = null;

		/**
		 * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态
		 */
		static final int CANCELLED =  1;
		/**
		 * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
		 */
		static final int SIGNAL    = -1;
		/**
		 * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
		 */
		static final int CONDITION = -2;
		/**
		 * 表示下一次共享式同步状态获取，将会无条件地传播下去
		 */
		static final int PROPAGATE = -3;

		/** 等待状态 */
		volatile int waitStatus;

		/** 前驱节点，当节点添加到同步队列时被设置（尾部添加） */
		volatile Node prev;

		/** 后继节点 */
		volatile Node next;

		/** 等待队列中的后续节点。如果当前节点是共享的，那么字段将是一个 SHARED 常量，也就是说节点类型（独占和共享）和等待队列中的后续节点共用同一个字段 */
		Node nextWaiter;

		/** 获取同步状态的线程 */
		volatile Thread thread;

		final boolean isShared() {
			return nextWaiter == SHARED;
		}

		final Node predecessor() throws NullPointerException {
			Node p = prev;
			if (p == null)
				throw new NullPointerException();
			else
				return p;
		}

		Node() { // Used to establish initial head or SHARED marker
		}

		Node(Thread thread, Node mode) { // Used by addWaiter
			this.nextWaiter = mode;
			this.thread = thread;
		}

		Node(Thread thread, int waitStatus) { // Used by Condition
			this.waitStatus = waitStatus;
			this.thread = thread;
		}

	}

predecessor




# addWaiter

入列

需要考虑并发的情况。它通过 CAS 的方式，来保证正确的添加 Node


	private Node addWaiter(Node mode) {
	   // 新建节点
	   Node node = new Node(Thread.currentThread(), mode);
	   // 记录原尾节点
	   Node pred = tail;
	   // 快速尝试，添加新节点为尾节点
	   if (pred != null) {
		   // 设置新 Node 节点的尾节点为原尾节点
		   node.prev = pred;
		   // CAS 设置新的尾节点
		   if (compareAndSetTail(pred, node)) {
			   // 成功，原尾节点的下一个节点为新节点
			   pred.next = node;
			   return node;
		   }
	   }
	   // 失败，多次尝试，直到成功
	   enq(node);
	   return node;
	}


	private Node enq(final Node node) {
		// 多次尝试，直到成功为止
		for (;;) {
			// 记录原尾节点
			Node t = tail;
			// 原尾节点不存在，创建首尾节点都为 new Node()
			if (t == null) {
				if (compareAndSetHead(new Node()))
					tail = head;
			// 原尾节点存在，添加新节点为尾节点
			} else {
				//设置为尾节点
				node.prev = t;
				// CAS 设置新的尾节点
				if (compareAndSetTail(t, node)) {
					// 成功，原尾节点的下一个节点为新节点
					t.next = node;
					return t;
				}
			}
		}
	}


# setHead

出列

private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}







   