

LockSupport提供了最基本的线程阻塞和唤醒功能



LockSupport定义了一组以park开头的方法用来阻塞当前线程，以及unpark(Thread thread)
方法来唤醒一个被阻塞的线程。



LockSupport 类与每 使用它的线程都会关联一 个许可证，在 默认情况下调用
LockSupport 类的方法的线程是不持有许可证的。 LockSupport 是使用 Unsafe 类实现的，



park

1. void pa （）方法
如果调用 park 方法的线程已经拿到了与 ockSupport 关联的许可证，则 调用
Locks up port. park（）时会马上返回，否则调用线程会被禁 参与线程的调度， 也就是会被阻
塞挂起。


在其他线程调用 unpark(Threa thread ）方法并且将当前线程作为参数时 ，调用 park
法而被阻塞的线程会返回。另外，如果其他线程调用了阻塞线程的 in terrupt （）方 ，设
了中断标志或者线程被虚假唤醒，则阻塞线程 会返回。所以在调用 park 方法时最好也
使用循环条件判断方式。
需要注意的是，因调用 rk （） 方法而被阻塞的线程被其 线程中断而返回 并不会抛
In terrupted Exception 异常。

unpark

void unpark(Thread thread ）方法
当一个线程调用 unpark 时，如果参数 thread 线程没有持有 thread ockSupport
关联的许可证， thread 线程持有。 如果 thread 之前因调用 park（）而被挂起，则调用
unpark 后，该线程会被唤醒 如果 thread 之前没有调用 park ，则 调用 unpark 方法后
调用 park 方法，其会立刻返 。



park(Object blocke 「
void parkNanos(long nos ）



 FIFOMutex 
 
 



# 来自 <Java并发编程之美>




