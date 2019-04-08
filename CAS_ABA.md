
解决方案 AtomicStampedReference
CAS 的 ABA 隐患问题，解决方案则是版本号，Java 提供了 AtomicStampedReference 来解决。
AtomicStampedReference 通过包装 [E,Integer] 的元组，来对对象标记版本戳 stamp ，从而避免 ABA 问题。对于上面的案例，应该线程 1 会失败。

AtomicStampedReference 的 #compareAndSet(...) 方法，代码如下：

public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}


## Pair

Pair 为 AtomicStampedReference 的内部类，主要用于记录引用和版本戳信息（标识），定义如下：

// AtomicStampedReference.java

private static class Pair<T> {
    final T reference; // 对象引用
    final int stamp; // 版本戳
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;


Pair 记录了对象的引用和版本戳，版本戳为 int 型，保持自增。同时 Pair 是一个不可变对象，其所有属性全部定义为 final 。
对外提供一个 #of(T reference, int stamp) 方法，该方法返回一个新建的 Pair 对象。
pair 属性，定义为 volatile ，保证多线程环境下的可见性。
在AtomicStampedReference 中，大多方法都是通过调用 Pair 的 #of(T reference, int stamp) 方法，来产生一个新的 Pair 对象，然后赋值给变量 pair 。
如 set(V newReference, int newStamp)方法，代码如下：

// AtomicStampedReference.java
public void set(V newReference, int newStamp) {
    Pair<V> current = pair;
    if (newReference != current.reference || newStamp != current.stamp)
        this.pair = Pair.of(newReference, newStamp);
}






# 实例

实际案例
下面，我们将通过一个例子，可以看到 AtomicStampedReference 和 AtomicInteger的区别。
我们定义两个线程，线程 1 负责将 100 —> 110 —> 100，线程 2 执行 100 —>120 ，看两者之间的区别。



	public class Test {
		private static AtomicInteger atomicInteger = new AtomicInteger(100);
		private static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(100,1);

		public static void main(String[] args) throws InterruptedException {

			// AtomicInteger
			Thread at1 = new Thread(new Runnable() {
				@Override
				public void run() {
					atomicInteger.compareAndSet(100,110);
					atomicInteger.compareAndSet(110,100);
				}
			});

			Thread at2 = new Thread(new Runnable() {
				@Override
				public void run() {
					try {
						TimeUnit.SECONDS.sleep(2);      // at1,执行完
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("AtomicInteger:" + atomicInteger.compareAndSet(100,120));
				}
			});

			at1.start();
			at2.start();

			at1.join();
			at2.join();

			// AtomicStampedReference

			Thread tsf1 = new Thread(new Runnable() {
				@Override
				public void run() {
					try {
						//让 tsf2先获取stamp，导致预期时间戳不一致
						TimeUnit.SECONDS.sleep(2);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					// 预期引用：100，更新后的引用：110，预期标识getStamp() 更新后的标识getStamp() + 1
					atomicStampedReference.compareAndSet(100,110,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);
					atomicStampedReference.compareAndSet(110,100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);
				}
			});

			Thread tsf2 = new Thread(new Runnable() {
				@Override
				public void run() {
					int stamp = atomicStampedReference.getStamp();

					try {
						TimeUnit.SECONDS.sleep(2);      //线程tsf1执行完
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("AtomicStampedReference:" +atomicStampedReference.compareAndSet(100,120,stamp,stamp + 1));
				}
			});

			tsf1.start();
			tsf2.start();
		}

	}




# 参考

http://www.iocoder.cn/JUC/sike/CAS/




