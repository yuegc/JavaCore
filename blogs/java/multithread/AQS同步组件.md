[toc]

## 一. ReentrantLock(重入锁)

**`ReentrantLock`**是对**排它锁**的实现，底层基于AQS，它又分为**公平锁**和**非公平锁**。

- 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁(**排队拿锁**)
- 非公平锁：当线程要获取锁时，先通过**两次 CAS** 操作去抢锁，如果没抢到，当前线程再加入到队列中等待唤醒(**先抢再排队**)

**`ReentrantLock` 默认采用非公平锁**(性能更好)，通过 boolean 来决定是否用公平锁

```java
private final Sync sync;
public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 1.1 非公平锁

```java
static final class NonfairSync extends Sync {
    final void lock() {
      	// 第一次CAS
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
          	// 调用AQS的acquire方法
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      	// 第二次CAS
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 1.2 公平锁

```java
static final class FairSync extends Sync {
  	// 公平锁不会进行CAS，直接调用AQS的acquire方法
    final void lock() {
        acquire(1);
    }

    
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
          	// hasQueuedPredecessors判断同步队列中是否有线程在排队
          	// 没有排队，则CAS尝试获取锁，非公平锁没有判断操作
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平锁和非公平锁的不同点(多了两次CAS)：

1. 非公平锁在调用 lock 后，首先 CAS 进行一次抢锁，如果获取到锁就直接返回
2. 非公平锁在第一次 CAS 失败后，和公平锁一样进入 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁被释放了(state == 0)，非公平锁会直接 CAS 抢锁，但公平锁则会判断同步队列是否有线程在排队，如果有则不抢锁，乖乖排到队列最后面。

**注：如果非公平锁两次 CAS 都不成功，那和公平锁是一样，都会进入到同步队列排队等待唤醒。**

相对来说，非公平锁会有更好的性能，因为它的吞吐量更大。但是，非公平锁让获取锁的时间变得更加不确定，可能会导致在同步中的线程长期处于饥饿状态。

### 1.3 ReentrantLock和Synchronized区别

`ReentrantLock`和`Synchronized`都是可重入锁，`synchronized` 是依赖于 JVM 实现的，而`ReentrantLock` 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成），那它们在功能上有哪些差别呢？

- **等待可中断** : `ReentrantLock`提供了一种能够中断等待锁的线程的机制，通过 `lock.lockInterruptibly()` 来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
- **可实现公平锁** : `ReentrantLock`可以指定是公平锁还是非公平锁。而`synchronized`只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。`ReentrantLock`默认情况是非公平的，可以通过 `ReentrantLock`类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
- **可实现选择性通知（锁可以绑定多个条件）**: `synchronized`关键字与`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。`ReentrantLock`类当然也可以实现，但是需要借助于`Condition`接口与`newCondition()`方法。

## 二. ReentrantReadWriteLock(读写锁)

**`ReentrantReadWriteLock`**是基于AQS实现的，它表示两个锁，**一个是读操作相关的锁，称为共享锁；一个是写相关的锁，称为排他锁**。适合**读多写少**的场景。

进入读锁的条件：

- 没有其他线程的写锁

- 没有其他线程的写请求

进入写锁的条件：

- 没有其他线程的读锁
- 没有其他线程的写锁

读写锁的三个重要特性：

​	**公平选择性**：支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平

​	**重进入**：读锁和写锁都支持线程重进入

​	**锁降级**：遵循获取写锁、获取读锁再释放写锁的次序，**写锁能够降级成为读锁，但读锁不能升级为写锁**

### 2.1 内部类

**`HoldCounter`**与读锁配套使用

```java
// 计数器
static final class HoldCounter {
  	// 读线程重入的次数
    int count = 0;
    // Use id, not reference, to avoid garbage retention
  	// 当前线程的TID属性的值
    final long tid = getThreadId(Thread.currentThread());
}
```

**`ThreadLocalHoldCounter`**

```java
// 本地线程计数器
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    // 重写初始化方法，在没有进行set的情况下，获取的都是该HoldCounter值
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

### 2.2 类属性

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 版本序列号
    private static final long serialVersionUID = 6317671515068378041L;        
    // 高16位为读锁，低16位为写锁
    static final int SHARED_SHIFT   = 16;
    // 读锁单位
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    // 读锁最大数量
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    // 写锁最大数量
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    // 本地线程计数器
    private transient ThreadLocalHoldCounter readHolds;
    // 缓存的计数器
    private transient HoldCounter cachedHoldCounter;
    // 第一个读线程(为了效率引入此变量，firstReader是不会放入到readHolds中的，当读锁仅有一个的情况下就会避免查找readHolds)
    private transient Thread firstReader = null;
    // 第一个读线程的计数
    private transient int firstReaderHoldCount;
}
```

### 2.3 构造函数

```java
// 构造函数
Sync() {
    // 本地线程计数器
    readHolds = new ThreadLocalHoldCounter();
    // 设置AQS的状态
    setState(getState()); // ensures visibility of readHolds
}
```

### 2.4 读写状态的设计

同步状态在重入锁的实现中是表示被同一个线程重复获取的次数，但是仅仅表示是否锁定，而不用区分是读锁还是写锁。而读写锁需要在同步状态（一个整形变量）上维护多个读线程和一个写线程的状态。

**读写锁对于同步状态的实现是在一个整形变量上通过“按位切割使用”：将变量切割成两部分，高16位表示读，低16位表示写。**

### 2.5 源码解析

#### 2.5.1 写锁的获取和释放

独占式同步状态的获取与释放实现的是Sync的 tryAcquire和 tryRelease

##### 2.5.1.1 tryAcquire(int acquires)

```java
protected final boolean tryAcquire(int acquires) {
    //当前线程
    Thread current = Thread.currentThread();
    //获取状态
    int c = getState();
    //写线程数量（即获取独占锁的重入数）
    int w = exclusiveCount(c);

    //当前同步状态state != 0，说明已经有其他线程获取了读锁或写锁
    if (c != 0) {
        // 当前state不为0，此时：如果写锁状态为0说明读锁此时被占用返回false；
        // 如果写锁状态不为0且写锁没有被当前线程持有返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;

        //判断同一线程获取写锁是否超过最大次数（65535），支持可重入
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //更新状态
        //此时当前线程已持有写锁，现在是重入，所以只需要修改锁的数量即可。
        setState(c + acquires);
        return true;
    }

    //到这里说明此时c=0,读锁和写锁都没有被获取
    //writerShouldBlock表示是否阻塞(跟reentrantLock公平、非公平锁相似)
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;

    //设置锁为当前线程所有
    setExclusiveOwnerThread(current);
    return true;
}
```

##### 2.5.1.2 tryRelease(int releases)

```java
protected final boolean tryRelease(int releases) {
    //若锁的持有者不是当前线程，抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //写锁的新线程数
    int nextc = getState() - releases;
    //如果独占模式重入数为0了，说明独占模式被释放
  	// exclusiveCount(nextc)表示占有写锁的线程数量
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        //若写锁的新线程数为0，则将锁的持有者设置为null
        setExclusiveOwnerThread(null);
    //设置写锁的新线程数
    //不管独占模式是否被释放，更新独占重入数
    setState(nextc);
    return free;
}
```

#### 2.5.2 读锁的获取和释放

共享式同步状态的获取与释放实现的是Sync的 tryAcquireShared 和 tryReleaseShared

##### 2.5.2.1 tryAcquireShared(int unused)

```java
protected final int tryAcquireShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();

    //如果写锁线程数 != 0 ，且独占锁不是当前线程则返回失败，因为存在锁降级
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 读锁数量
    int r = sharedCount(c);
    /*
     * readerShouldBlock():读锁是否需要等待（公平锁原则）
     * r < MAX_COUNT：持有线程小于最大数（65535）
     * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
     */
     // 读线程是否应该被阻塞、并且小于最大值、并且比较设置成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        //r == 0，表示第一个读锁线程，第一个读锁firstRead是不会加入到readHolds中
        if (r == 0) { // 读锁数量为0
            // 设置第一个读线程
            firstReader = current;
            // 读线程占用的资源数为1
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 当前线程为第一个读线程，表示第一个读锁线程重入
            // 占用资源数加1
            firstReaderHoldCount++;
        } else { // 读锁数量不为0并且不为当前线程
            // 获取计数器
            HoldCounter rh = cachedHoldCounter;
            // 计数器为空或者计数器的tid不为当前正在运行的线程的tid
            if (rh == null || rh.tid != getThreadId(current))
                // 获取当前线程对应的计数器
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0) // 计数为0
                //加入到readHolds中
                readHolds.set(rh);
            //计数+1
            rh.count++;
        }
        return 1;
    }
  	// 如果不满足 读线程是否应该被阻塞、并且小于最大值、并且比较设置成功
    return fullTryAcquireShared(current);
}
```

###### 2.5.2.1.1 fullTryAcquireShared(Thread current)

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) { // 无限循环
        // 获取状态
        int c = getState();
        if (exclusiveCount(c) != 0) { // 写线程数量不为0
            if (getExclusiveOwnerThread() != current) // 不为当前线程
                return -1;
        } else if (readerShouldBlock()) { // 写线程数量为0并且读线程被阻塞
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) { // 当前线程为第一个读线程
                // assert firstReaderHoldCount > 0;
            } else { // 当前线程不为第一个读线程
                if (rh == null) { // 计数器不为空
                    // 
                    rh = cachedHoldCounter;
                  	// 计数器为空或者计数器的tid不为当前正在运行的线程的tid
                    if (rh == null || rh.tid != getThreadId(current)) { 
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT) // 读锁数量为最大值，抛出异常
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // 比较并且设置成功
            if (sharedCount(c) == 0) { // 读线程数量为0
                // 设置第一个读线程
                firstReader = current;
                // 
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

##### 2.5.2.2 tryReleaseShared(int)

```java
protected final boolean tryReleaseShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    if (firstReader == current) { // 当前线程为第一个读线程
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1) // 读线程占用的资源数为1
            firstReader = null;
        else // 减少占用的资源
            firstReaderHoldCount--;
    } else { // 当前线程不为第一个读线程
        // 获取缓存的计数器
        HoldCounter rh = cachedHoldCounter;
      	// 计数器为空或者计数器的tid不为当前正在运行的线程的tid
        if (rh == null || rh.tid != getThreadId(current)) 
            // 获取当前线程对应的计数器
            rh = readHolds.get();
        // 获取计数
        int count = rh.count;
        if (count <= 1) { // 计数小于等于1
            // 移除
            readHolds.remove();
            if (count <= 0) // 计数小于等于0，抛出异常
                throw unmatchedUnlockException();
        }
        // 减少计数
        --rh.count;
    }
    for (;;) { // 无限循环
        // 获取状态
        int c = getState();
        // 获取状态
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc)) // 比较并进行设置
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

在读锁的获取、释放过程中，总是会有一个对象存在着，同时该对象在获取线程获取读锁是+1，释放读锁时-1，该对象就是HoldCounter。

  要明白HoldCounter就要先明白读锁。前面提过读锁的内在实现机制就是共享锁，对于共享锁其实我们可以稍微的认为它不是一个锁的概念，它更加像一个计数器的概念。一次共享锁操作就相当于一次计数器的操作，获取共享锁计数器+1，释放共享锁计数器-1。只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以**HoldCounter的作用就是当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。**

## 三. Semaphore(信号量)

**`Semaphore`**是共享锁的一种实现，它默认构造AQS的state为permits。可以指定**多个线程同时访问某个资源**，用于那些**资源有明确访问数量限制的场景**，常用于**限流**，如数据库连接池等。

当执行任务的线程数量超出permits,那么多余的线程将会被放入阻塞队列Park,并自旋判断state是否大于0。只有当state大于0的时候，阻塞的线程才能继续执行,此时先前执行任务的线程继续执行release方法，release方法使得state的变量会加1，那么自旋的线程便会判断成功。如此，每次只有最多不超过permits数量的线程能自旋成功，便限制了执行任务线程的数量。

### 3.1 构造器

```java
// 默认非公平
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

### 3.2 源码解析

Semaphore 有 2 个重要的方法，也是我们经常使用的 2 个方法：

```java
semaphore.acquire();
// doSomeing.....
semaphore.release();
```

#### 3.2.1 acquire()/acquire(int)

```java
public void acquire() throws InterruptedException {
		// 尝试获取一个锁
    sync.acquireSharedInterruptibly(1);
}

public void acquire(int permits) throws InterruptedException {
  	// 多一步判断，如果请求的许可小于0抛异常
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

##### 3.2.1.1 acquireSharedInterruptibly(int)

```java
// 这是抽象类 AQS 的方法
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 如果小于0，就获取锁失败了。加入到AQS 等待队列中。
    // 如果大于0，就直接执行下面的逻辑了。不用进行阻塞等待。
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

**`tryAcquireShared`**请参考[AQS原理3.4](AQS原理)。这里说一下doAcquireSharedInterruptibly，该方法跟AQS原理中3.4.1[doAcquireShared](AQS原理)一样，只不过这里会抛出异常而已。

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 添加一个节点 AQS 队列尾部
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
    	// 死循环
        for (;;) {
        	// 找到新节点的上一个节点
            final Node p = node.predecessor();
            // 如果这个节点是 head，就尝试获取锁
            if (p == head) {
            	// 继续尝试获取锁，这个方法是子类实现的
                int r = tryAcquireShared(arg);
                // 如果大于0，说明拿到锁了。
                if (r >= 0) {
                	// 将 node 设置为 head 节点
                	// 如果大于0，就说明还有机会获取锁，那就唤醒后面的线程，称之为传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 如果他的上一个节点不是 head，就不能获取锁
            // 对节点进行检查和更新状态，如果线程应该阻塞，返回 true。
            if (shouldParkAfterFailedAcquire(p, node) &&
            	// 阻塞 park，并返回是否中断，中断则抛出异常
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
        	// 取消节点
            cancelAcquire(node);
    }
}
```

### 3.2.2 release()/release(int)

```java
public void release() {
    sync.releaseShared(1);
}
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```

**`releaseShared`**请参考[AQS原理3.5](AQS原理)，这里不再赘述。

## 四. CountDownLatch(闭锁)

**CountDownLatch允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。**

**`CountDownLatch`**也是是共享锁的一种实现,它默认构造 AQS 的 state 值为 count。当线程使用countDown方法时,其实使用了`tryReleaseShared`方法以CAS的操作来减少state,直至state为0就代表所有的线程都调用了countDown方法。当调用await方法的时候，如果state不为0，就代表仍然有线程没有调用countDown方法，那么就把已经调用过countDown的线程都放入阻塞队列Park,并自旋CAS判断state == 0，直至最后一个线程调用了countDown，使得state == 0，于是阻塞的线程便判断成功，全部往下执行。

### 4.1 两种典型用法

1. 某一线程在开始运行前等待 n 个线程执行完毕。将 CountDownLatch 的计数器初始化为 n ：`new CountDownLatch(n)`，每当一个任务线程执行完毕，就将计数器减 1 `countdownlatch.countDown()`，当计数器的值变为 0 时，在`CountDownLatch上 await()` 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。
2. 实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 `CountDownLatch` 对象，将其计数器初始化为 1 ：`new CountDownLatch(1)`，多个线程在开始执行任务前首先 `coundownlatch.await()`，当主线程调用 countDown() 时，计数器变为 0，多个线程同时被唤醒。

**不足：**CountDownLatch 是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当 CountDownLatch 使用完毕后，它不能再次被使用。

CountDownLatch的实现跟Semaphore很相似，不同在于对`tryAcquireShared`和`tryReleaseShared`的实现，有兴趣可以去看一下。

## 五. CyclicBarrier(循环栅栏)

CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。

CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。CyclicBarrier 默认的构造方法是 `CyclicBarrier(int parties)`，其参数表示屏障拦截的线程数量，每个线程调用`await`方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。

### 5.1 构造函数：

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

// 线程到达屏障时，优先执行 barrierAction
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

其中，parties 就代表了有拦截的线程的数量，当拦截的线程数量达到这个值的时候就打开栅栏，让所有线程通过。

#### 5.1 CyclicBarrier 的应用场景

CyclicBarrier 可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个 Excel 保存了用户所有银行流水，每个 Sheet 保存一个帐户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个 sheet 里的银行流水，都执行完之后，得到每个 sheet 的日均银行流水，最后，再用 barrierAction 用这些线程的计算结果，计算出整个 Excel 的日均银行流水。

#### 5.2 CyclicBarrier 的使用示例

示例 1：

```java
public class CyclicBarrierDemo {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    try {
      /**等待60秒，保证子线程完全执行结束*/
      cyclicBarrier.await(60, TimeUnit.SECONDS);
    } catch (Exception e) {
      System.out.println("-----CyclicBarrierException------");
    }
    System.out.println("threadnum:" + threadnum + "is finish");
  }

}
```

运行结果，如下：

```
threadnum:0is ready
threadnum:1is ready
threadnum:2is ready
threadnum:3is ready
threadnum:4is ready
threadnum:4is finish
threadnum:0is finish
threadnum:1is finish
threadnum:2is finish
threadnum:3is finish
threadnum:5is ready
threadnum:6is ready
threadnum:7is ready
threadnum:8is ready
threadnum:9is ready
threadnum:9is finish
threadnum:5is finish
threadnum:8is finish
threadnum:7is finish
threadnum:6is finish
......
```

可以看到当线程数量也就是请求数量达到我们定义的 5 个的时候， `await`方法之后的方法才被执行。

另外，CyclicBarrier 还提供一个更高级的构造函数`CyclicBarrier(int parties, Runnable barrierAction)`，用于在线程到达屏障时，优先执行`barrierAction`，方便处理更复杂的业务场景。示例代码如下：

```java
public class CyclicBarrierDemo {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
    System.out.println("------当线程数达到之后，优先执行------");
  });

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          // TODO Auto-generated catch block
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum:" + threadnum + "is ready");
    cyclicBarrier.await();
    System.out.println("threadnum:" + threadnum + "is finish");
  }

}
```

运行结果，如下：

```
threadnum:0is ready
threadnum:1is ready
threadnum:2is ready
threadnum:3is ready
threadnum:4is ready
------当线程数达到之后，优先执行------
threadnum:4is finish
threadnum:0is finish
threadnum:2is finish
threadnum:1is finish
threadnum:3is finish
threadnum:5is ready
threadnum:6is ready
threadnum:7is ready
threadnum:8is ready
threadnum:9is ready
------当线程数达到之后，优先执行------
threadnum:9is finish
threadnum:5is finish
threadnum:6is finish
threadnum:8is finish
threadnum:7is finish
......
```

#### 5.3 CyclicBarrier源码分析

当调用 `CyclicBarrier` 对象调用 `await()` 方法时，实际上调用的是`dowait(false, 0L)`方法。 `await()` 方法就像树立起一个栅栏的行为一样，将线程挡住了，当拦住的线程数量达到 parties 的值时，栅栏才会打开，线程才得以通过执行。

```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
```

`dowait(false, 0L)`：

```java
    // 当线程数量或者请求数量达到 count 时 await 之后的方法才会被执行。上面的示例中 count 的值就为 5。
    private int count;
    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        // 锁住
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            // 如果线程中断了，抛出异常
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            // cout减1
            int index = --count;
            // 当 count 数量减为 0 之后说明最后一个线程已经到达栅栏了，也就是达到了可以执行await 方法之后的条件
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    // 将 count 重置为 parties 属性的初始化值
                    // 唤醒之前等待的线程
                    // 下一波执行开始
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

```

总结：`CyclicBarrier` 内部通过一个 count 变量作为计数器，cout 的初始值为 parties 属性的初始化值，每当一个线程到了栅栏这里了，那么就将计数器减一。如果 count 值为 0 了，表示这是这一代最后一个线程到达栅栏，就尝试执行我们构造方法中输入的任务。

#### 5.4 CyclicBarrier 和 CountDownLatch 的区别

- CountDownLatch的实现是基于AQS的，而CycliBarrier是基于 ReentrantLock(ReentrantLock也属于AQS同步器)和 Condition 的。

- CountDownLatch 是计数器，只能使用一次，而 CyclicBarrier 的计数器提供 reset 功能，可以多次使用。


- 对于 CountDownLatch 来说，重点是“一个线程（多个线程）等待”，而其他的 N 个线程在完成“某件事情”之后，可以终止，也可以等待。而对于 CyclicBarrier，重点是多个线程，在任意一个线程没有完成，所有的线程都必须等待。


- CountDownLatch 是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而 CyclicBarrier 更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。