 synchronized
 
 在多线程并发编程中synchronized一直是元老级角色，很多人都会称呼它为重量级锁。但是，随着Java SE 1.6对synchronized进行了各种优化之后，有些情况下它就并不那么重了。本文详细介绍Java SE 1.6中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁，以及锁的存储结构和升级过
 
 
 # 锁
 
 当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。
 
 
 先来看下利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁。
 具以下3种形式。·对于普通同步方法，锁是当前实例对象。
 ·对于静态同步方法，锁是当前类的Class对象。
 ·对于同步方法块，锁是Synchonized括号里配置的
 
 # 指令
 
 monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处，
任何一个monitor都有对象与之关联，
线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。
当且一个monitor被持有后，它将处于锁定状态。


# 对象

synchronized用的锁是存在Java对象头里的Mark Word里
在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化


# 锁的升级与对比

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在
Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状
态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏
向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高
获得锁和释放锁的效率，下

# 1.偏向锁
大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。
当一个线程访问同步块并
获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出
同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否
存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。

如果测试失败，则需
要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则
使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程

（1）偏向锁的撤销
偏向锁使用了一种等到竞争出现才释放锁的机制，
所以当其他线程尝试竞争偏向锁时，
持有偏向锁的线程才会释放锁。
偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正
在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，
如果线程不处于活动状态，则将对象头设置成无锁状态；
如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他
线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。


# 2.轻量级锁
（1）轻量级锁加锁
线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并
将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用
CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失
败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁

（2）轻量级锁解锁
轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成
功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。图2-2是
两个线程同时争夺锁，导致锁膨胀的流程图。





