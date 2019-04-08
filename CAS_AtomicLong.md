
# 概述

  AtomicLong 是原子性递增或者递减类，其内部使用 Unsafe 来实现
  
  
# 类图

private static final Unsafe unsafe = Unsafe . getUnsafe () ; 

private volatile long value ; 

private static final long valueOff set; 


# 实现

递增和递减操作

在如上代码内部都是通过调用 nsafe getAndAddLong 方法来实现操作，这个函数
是个原子性操作，这里第一个参数是 Ato micLong 实例的引用 第二个参数是 value 变量
AtomicLong 中的偏移值，第三个参数是要设置的第二个变量的值。

其中， getAndlncrement 方法在 JDK 中的实现逻辑
publi c final long getAndincremeηt () { 
while (true) { 
long current= get() ; 
long next = current + l ; 
if (compareAndSet(current , next)) 
return current; 
在如上代码中，每个线程是先拿到变量的当前值（由于 va lue volatile 变量，所以这
里拿到的是最新的值），然后在工作内存中对其进行增 操作 ，而后使用 CAS 修改变量 值。
如果设置失败，则循环继续尝试 到设置成功



2. boolean compareAndSet(long expect, long update ）方
ubl final boolean ompareAndSet(lo口g expect , long update) { 
return unsafe compareA dSwapLong this valueOffset , e xpect , update) ; 
由如上代码可知，在内部还 调用了 nsafe.co pareAnd SwapLo 如果原
量中的 va 值等于 ex ect ，则使用 ate 值更新 值并返回 true 否则返回 fa se








