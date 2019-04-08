Lock 接口
java.util.concurrent.locks.Lock 接口，定义方法如下：

void lock();
void lockInterruptibly() throws InterruptedException;
boolean tryLock();
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

void unlock();

Condition newCondition();
