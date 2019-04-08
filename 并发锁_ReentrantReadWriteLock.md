java.util.concurrent.locks.ReentrantReadWriteLock ，实现 ReadWriteLock 接口，可重入的读写锁实现类。在它内部，维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 Writer 线程，读取锁可以由多个 Reader 线程同时保持。也就说说，写锁是独占的，读锁是共享的。
