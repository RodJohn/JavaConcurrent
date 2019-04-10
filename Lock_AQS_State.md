

对于 AQS 来说，线程同步的关键是对状态值 state 进行操作


AQS 使用一个 int 类型的成员变量 state 来表示同步状态：

当 state > 0 时，表示已经占用锁。
当 state = 0 时，表示已经释放锁。


对于 Reentran tLock 现来说， state 可以用 来表示
前线程获取锁的可重入次数 ；对于 读写锁 ReentrantReadWri teLock 来说 state 16
位表示读状态，也就是获取该读锁的次数，低 16 位表示获取到写锁的线程的可重入次数；
对于 semaphore 来说， state 用来表示当前可用信号的 数：对于 CountDownlatch 来说，
state 来表示 数器当前的值


它提供了三个方法，来对同步状态 state 进行操作，并且 AQS 可以确保对 state 的操作是安全的：

#getState()
#setState(int newState)
#compareAndSetState(int expect, int update)








