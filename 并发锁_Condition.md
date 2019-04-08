# 作用

在没有 Lock 之前，我们使用 synchronized 来控制同步，配合 Object 的 #wait()、#notify() 等一系列方法可以实现等待 / 通知模式。
在 Java SE 5 后，Java 提供了 Lock 接口，相对于 synchronized 而言，Lock 提供了条件 Condition ，对线程的等待、唤醒操作更加详细和灵活。
下图是 Condition 与 Object 的监视器方法的对比

Condition 是一种广义上的条件队列。
他为线程提供了一种更为灵活的等待 / 通知模式，
线程在调用 await 方法后执行挂起操作，直到线程等待的某个条件为真时才会被唤醒。
Condition 必须要配合 Lock 一起使用，因为对共享状态变量的访问发生在多线程环境下。
一个 Condition 的实例必须与一个 Lock 绑定，因此 Condition 一般都是作为 Lock 的内部实现。



一个线程获取锁后，
通过调用 Condition 的 #await() 方法，
会将当前线程先加入到条件队列中，然后释放锁，
最后通过 #isOnSyncQueue(Node node) 方法，不断自检看节点是否已经在 CLH 同步队列了，如果是则尝试获取锁，否则一直挂起。

当线程调用 #signal() 方法后，
程序首先检查当前线程是否获取了锁，
然后通过#doSignal(Node first) 方法唤醒CLH同步队列的首节点。
被唤醒的线程，将从 #await() 方法中的 while 循环中退出来，
然后调用 #acquireQueued(Node node, int arg) 方法竞争同步状态。



# ConditionObject




# await

调用 Condition 的 #await() 方法，
会使当前线程进入等待状态，同时会加入到 Condition 等待队列，并且同时释放锁。
当从 #await() 方法结束时，当前线程一定是获取了Condition 相关联的锁。


	public final void await() throws InterruptedException {
		// 当前线程中断
		if (Thread.interrupted())
			throw new InterruptedException();
		//当前线程加入等待队列
		Node node = addConditionWaiter();
		//释放锁
		long savedState = fullyRelease(node);
		int interruptMode = 0;
		/**
		 * 检测此节点的线程是否在同步队上，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待
		 * 直到检测到此节点在同步队列上
		 */
		while (!isOnSyncQueue(node)) {
			//线程挂起
			LockSupport.park(this);
			//如果已经中断了，则退出
			if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
				break;
		}
		//竞争同步状态
		if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
			interruptMode = REINTERRUPT;
		// 清理下条件队列中的不是在等待条件的节点
		if (node.nextWaiter != null) // clean up if cancelled
			unlinkCancelledWaiters();
		if (interruptMode != 0)
			reportInterruptAfterWait(interruptMode);
	}

首先，将当前线程新建一个节点同时加入到条件队列中。
然后，释放当前线程持有的同步状态。
之后，则是不断检测该节点代表的线程，出现在 CLH 同步队列中（收到 signal 信号之后，就会在 AQS 队列中检测到），如果不存在则一直挂起。
最后，重新参与竞争，获取到同步状态。



## addConditionWaiter

	private Node addConditionWaiter() {
		Node t = lastWaiter;    //尾节点
		//Node的节点状态如果不为CONDITION，则表示该节点不处于等待状态，需要清除节点
		if (t != null && t.waitStatus != Node.CONDITION) {
			//清除条件队列中所有状态不为Condition的节点
			unlinkCancelledWaiters();
			t = lastWaiter;
		}
		//当前线程新建节点，状态 CONDITION
		Node node = new Node(Thread.currentThread(), Node.CONDITION);
		/**
		 * 将该节点加入到条件队列中最后一个位置
		 */
		if (t == null)
			firstWaiter = node;
		else
			t.nextWaiter = node;
		lastWaiter = node;
		return node;
	}



该方法主要是将当前线程加入到 Condition 条件队列中。
当然，在加入到尾节点之前，会调用 #unlinkCancelledWaiters() 方法，清除所有状态不为 Condition 的节点。


## fullyRelease


负责完全释放该线程持有的锁，因为例如 ReentrantLock 是可以重入的。

	final long fullyRelease(Node node) {
		boolean failed = true;
		try {
			// 节点状态--其实就是持有锁的数量
			long savedState = getState();
			// 释放锁
			if (release(savedState)) {
				failed = false;
				return savedState;
			} else {
				throw new IllegalMonitorStateException();
			}
		} finally {
			if (failed)
				node.waitStatus = Node.CANCELLED;
		}
	}

正常情况下，释放锁都能成功，因为是先调用 Lock#lock() 方法，再调用 Condition#await() 方法。
那么什么情况下会失败，抛出 IllegalMonitorStateException 异常呢？
例如，当前线程未持有锁，未调用 Lock#lock() 方法，而直接调用 Condition#await() 方法，此时就会抛出该异常。
另外，释放失败的情况下，会设置 Node 的等待状态为 Node.CANCELED 。

## isOnSyncQueue

isOnSyncQueue(Node node) 方法，如果一个节点刚开始在条件队列上，现在在同步队列上获取锁则返回 true 。代码如下：

	final boolean isOnSyncQueue(Node node) {
		// 状态为 Condition，获取前驱节点为 null ，返回 false
		if (node.waitStatus == Node.CONDITION || node.prev == null)
			return false;
		// 后继节点不为 null，肯定在 CLH 同步队列中
		if (node.next != null)
			return true;

		return findNodeFromTail(node);
	}

## unlinkCancelledWaiters

#unlinkCancelledWaiters() 方法，负责将条件队列中状态不为 Condition 的节点删除。代码如下：

	// 等待队列是一个单向链表，遍历链表将已经取消等待的节点清除出去
	// 纯属链表操作，很好理解，看不懂多看几遍就可以了
	private void unlinkCancelledWaiters() {
		Node t = firstWaiter;
		Node trail = null; // 用于中间不需要跳过时，记录上一个 Node 节点
		while (t != null) {
			Node next = t.nextWaiter;
			// 如果节点的状态不是 Node.CONDITION 的话，这个节点就是被取消的
			if (t.waitStatus != Node.CONDITION) {
				t.nextWaiter = null;
				if (trail == null)
					firstWaiter = next;
				else
					trail.nextWaiter = next;
				if (next == null)
					lastWaiter = trail;
			}
			else
				trail = t;
			t = next;
		}
	}



# signal

调用 ConditionObject的 #signal() 方法，
将会唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到CLH同步队列中。

	public final void signal() {
		//检测当前线程是否为拥有锁的独
		if (!isHeldExclusively())
			throw new IllegalMonitorStateException();
		//头节点，唤醒条件队列中的第一个节点
		Node first = firstWaiter;
		if (first != null)
			doSignal(first);    //唤醒
	}


该方法首先会判断当前线程是否已经获得了锁，这是前置条件。然后调用 #doSignal(Node first) 方法，唤醒条件队列中的头节点。代码如下：

doSignal

private void doSignal(Node first) {
    do {
        //修改头结点，完成旧头结点的移出工作
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
            (first = firstWaiter) != null);
}
主要是做两件事：1）修改头节点；2）调用 #transferForSignal(Node first) 方法将节点移动到 CLH 同步队列中。代码如下：

 final boolean transferForSignal(Node node) {
    //将该节点从状态CONDITION改变为初始状态0,
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    //将节点加入到syn队列中去，返回的是syn队列中node节点前面的一个节点
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果结点p的状态为cancel 或者修改waitStatus失败，则直接唤醒
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
x
整个通知的流程如下：

判断当前线程是否已经获取了锁，如果没有获取则直接抛出异常，因为获取锁为通知的前置条件。
如果线程已经获取了锁，则将唤醒条件队列的首节点
唤醒首节点是先将条件队列中的头节点移出，然后调用 AQS 的 #enq(Node node) 方法将其安全地移到 CLH 同步队列中
最后判断如果该节点的同步状态是否为 Node.CANCEL ，或者修改状态为 Node.SIGNAL 失败时，则直接调用 LockSupport 唤醒该节点的线程。



# 示例


public class ConditionTest {
    private LinkedList<String> buffer;    //容器
    private int maxSize ;           //容器最大
    private Lock lock;
    private Condition fullCondition;
    private Condition notFullCondition;

    ConditionTest(int maxSize){
        this.maxSize = maxSize;
        buffer = new LinkedList<String>();
        lock = new ReentrantLock();
        fullCondition = lock.newCondition();
        notFullCondition = lock.newCondition();
    }

    public void set(String string) throws InterruptedException {
        lock.lock();    //获取锁
        try {
            while (maxSize == buffer.size()){
                notFullCondition.await();       //满了，添加的线程进入等待状态
            }

            buffer.add(string);
            fullCondition.signal();
        } finally {
            lock.unlock();      //记得释放锁
        }
    }

    public String get() throws InterruptedException {
        String string;
        lock.lock();
        try {
            while (buffer.size() == 0){
                fullCondition.await();
            }
            string = buffer.poll();
            notFullCondition.signal();
        } finally {
            lock.unlock();
        }
        return string;
    }
}











