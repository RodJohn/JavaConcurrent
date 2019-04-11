# 同步队列



作用

	当前线程如果获取同步状态失败时，
	AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，
	就会进入一个自旋的过程，每个节点都会自省地观察，
	
	
	当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。


# 结构


	QS IFO 的双向队列，其内部通过节点 head tail首和队尾元素，
	队列元素的类型为 ode 其中 Node 中的 thread 变量用来存放进入 AQS队列里面的线程：




# Node

	prev 记录当前节点的前驱节点， next 记录当前节点的后继节点

	SHARED 用来标记该线程是获取共 资源时被阻挂起后放入 QS 队列的， 
	EXCLUSIVE 用来标记线程是 取独占资源时被挂起后放入AQS 队列的

	waitStatu 记录当前线程等待状态，
	用来控制线程的阻塞和唤醒，并且可以避免不必要的调用LockSupport的 #park(...) 和 #unpark(...) 方法。
	不仅仅指的是 Node 自己的线程的等待状态，也可以是下一个节点的线程的等待状态。
	CANCELLED （线程被取消了）、
	SIGNAL 线程需要被唤醒）、 
	ONDITION （线程在条件队列里面等待〉
	PROPAGATE （释放共享资源时需要通知其他节点〕；
	INITAL ，值为 0 ，初始状态。


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

	通过CAS修改对象字段的方式，来保证并发下正确的添加 Node

流程

	将当前线程以及相应模式，构造成一个节点Node
	尾节点存在，直接通过CAS在同步队列尾部添加节点
	否则进入循环添加模式
	如果尾节点为空,则先添加一个空节点为头结点
	如果尾节点不为空，那么并发下使用CAS算法将当前Node追加成为尾节点


代码

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



## compareAndSetTail

作用

	通过CAS方法原子修改对象字段

代码

	private static final Unsafe unsafe = Unsafe.getUnsafe();

	private static final long tailOffset = unsafe.objectFieldOffset (AbstractQueuedSynchronizer.class.getDeclaredField("tail")); 

	private final boolean compareAndSetTail(Node expect, Node update) {
	    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
	}

## compareAndSetHead

代码

	private static final long headOffset = unsafe.objectFieldOffset (AbstractQueuedSynchronizer.class.getDeclaredField("head")); 

	private final boolean compareAndSetHead(Node update) {
	    return unsafe.compareAndSwapObject(this, headOffset, null, update);
	}


# setHead

出列

代码

	private void setHead(Node node) {
	    head = node;
	    node.thread = null;
	    node.prev = null;
	}







   
