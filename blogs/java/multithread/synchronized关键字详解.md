[toc]

## 一. synchronized作用

**`synchronized` 关键字解决的是多个线程之间访问资源的同步性，`synchronized`关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。**

> 在早期Java版本中**`synchronized`**属于重量级锁。为什么？
>
> 因为监视器锁（monitor）是依赖于底层的操作系统的 `Mutex Lock` 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高。
>
> JDK1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。现在JDK源码和很多开源框架里面都大量使用了sychronized。

## 二. 使用方式

**synchronized 关键字最主要的三种使用方式：**

**1.修饰实例方法:** 给当前对象实例加锁，进入同步代码前要获得 **当前对象实例的锁**

```java
synchronized void method(){
  ......
}
```

**2.修饰静态方法:** 也就是给当前类加锁，进入同步代码前要获得 **当前 class 对象的锁**。因为静态成员不属于任何一个实例对象，是类成员（ _static 表明这是该类的一个静态资源，不管 new 了多少个对象，只有一份_）。所以，如果一个线程 A 调用一个实例对象的非静态 `synchronized` 方法，而线程 B 需要调用这个实例对象所属类的静态 `synchronized` 方法，是允许的，不会发生互斥现象，**因为访问静态 `synchronized` 方法占用的锁是当前类对象的锁，而访问非静态 `synchronized` 方法占用的锁是当前实例对象的锁**。

```java
synchronized void staic method() {
  ........
}
```

**3.修饰代码块:** 指定加锁对象，对给定对象/类加锁。`synchronized(this|object)` 表示进入同步代码块前要获得**给定对象的锁**。`synchronized(类.class)` 表示进入同步代码前要获得 **当前 class 对象的锁**

```java
synchronized(this) {
  ....
}

synchronized(类.class) {
  .....
}
```

## 三. synchronized实现原理

上面讲到**`synchronized`**是对 **类对象** 或者 **实例对象** 加锁，那为什么这些对象都可以实现锁呢？

1. 首先，Java 中的每个对象都派生自 Object 类，而每个Java Object 在 JVM 内部都有一个 native 的 C++对象 oop/oopDesc 进行对应。
2. 线程在获取锁的时候，实际上就是获得一个**对象监视器 (monitor)** ，**`monitor` **可以认为是一个同步对象，所有的 Java 对象是天生携带 **`monitor`**。在 hotspot 源码的 markOop.hpp 文件中，可以看到下面这段代码。

<img src="../../resource/thread/monitor.png" alt="monitor" style="zoom:50%;" />

多个线程访问同步方法/代码块时，相当于去争抢对象监视器，修改**[对象头](#3.1-Java对象头)**中的**锁标识**,上面的代码中ObjectMonitor这个 对象和线程争抢锁的逻辑有密切的关系。

> `wait/notify`等方法也依赖于`monitor`对象，这就是为什么只有在同步的块或者方法中才能调用`wait/notify`等方法，否则会抛出`java.lang.IllegalMonitorStateException`的异常的原因。

ObjectMonitor主要数据结构如下：

```c++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //monitor进入数
    _waiters      = 0,
    _recursions   = 0;  //线程的重入次数
    _object       = NULL;
    _owner        = NULL; //标识拥有该monitor的线程
    _WaitSet      = NULL; //等待线程组成的双向循环链表，_WaitSet是第一个节点
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; //多线程竞争锁进入时的单项链表
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

- owner：初始时为NULL。当有线程占有该monitor时，owner标记为该线程的唯一标识。当线程释放monitor时，owner又恢复为NULL。owner是一个临界资源，JVM是通过CAS操作来保证其线程安全的
-  _cxq：竞争队列，所有请求锁的线程首先会被放在这个队列中（单向链接）。_cxq是一个临界资源，JVM通过CAS原子指令来修改_cxq队列。修改前_cxq的旧值填入了node的next字段，_cxq指向新值（新线程）。因此_cxq是一个后进先出的stack（栈）
-  EntryList：_cxq队列中有资格成为候选资源的线程会被移动到该队列中
-  WaitSet：因为调用wait方法而被阻塞的线程会被放在该队列中

### 3.1 Java对象头

在HotSpot虚拟机中, 对象在内存中的布局分为三块区域: **对象头**、**实例数据**和**对其填充**。

- **对象头：**由**MarkWord**、**Klass Point(类型指针)**和**数组长度(只有数组对象才有)**组成

  - **Klass Point：**是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例，该指针在32位JVM中的长度是32bit，在64位JVM中长度是64bit

  - **Mark Word：**用于存储对象自身的运行时数据，如HashCode, GC分代年龄, 锁状态标志, 线程持有的锁, 偏向线程ID等等。占用内存大小与虚拟机位长一致(32位JVM -> MarkWord是32位, 64位JVM->MarkWord是64位)

    <img src="../../resource/thread/objecthead.png" alt="image-20201120163107030" style="zoom:50%;" />

  - **数组长度：**只有数组对象保存了这部分数据，该数据在32位和64位JVM中长度都是32bit

  如果对象是数组对象，那么对象头占用3个字宽（Word），如果对象是非数组对象，那么对象头占用2个字宽。（1word = 2 Byte = 16 bit）

- **实例数据：**存储的是对象的属性信息，包括父类的属性信息，按照4字节对齐

- **填充字符：**因为虚拟机要求对象字节必须是8字节的整数倍，填充字符就是用于凑齐这个整数倍的

### 3.2 获取monitor对象

在JVM规范里可以看到，不管是方法同步还是代码块同步都是基于进入和退出monitor对象来实现，然而二者在具体实现上又存在很大的区别。通过javap对class字节码文件反编译可以得到反编译后的代码。

> 通过 JDK 自带的 `javap` 命令查看 `SyncDemo` 类的相关字节码信息：
>
> 首先切换到类的对应目录执行 `javac SyncDemo.java` 命令生成编译后的 .class 文件，然后执行`javap -c -s -v -l SyncDemo.class`。

#### 3.2.1 synchronized修饰同步代码块

```java
public class SyncDemo {
	public void method() {
		synchronized (this) {
			System.out.println("synchronized 代码块");
		}
	}
}
```

![image-20201120153509921](../../resource/thread/synchronized1.png)

**`synchronized` 同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。**

在执行`monitorenter`时，会尝试获取对象的锁，如果锁的计数器为 0 则表示锁可以被获取，获取后将锁计数器设为 1 也就是加 1。

在执行 `monitorexit` 指令后，将锁计数器设为 0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

#### 3.1.2 synchronized修饰方法

```java
public class SyncDemo {
	public synchronized void method() {
		System.out.println("synchronized 方法");
	}
}
```

![image-20201120153030172](../../resource/thread/synchronized2.png)

**`synchronized`** 修饰的方法并没有 **`monitorenter`** 指令和 **`monitorexit`** 指令，取得代之的确实是 **`ACC_SYNCHRONIZED`** 标识，该标识指明了该方法是一个同步方法。JVM 通过该 **`ACC_SYNCHRONIZED`** 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

**两者的本质都是对 对象监视器 monitor 的获取。**

## 四. 锁优化

### 4.1 锁的分类

锁主要存在四种状态，级别从低到高依次是：**无锁状态**、**偏向锁状态**、**轻量级锁状态**、**重量级锁状态**，他们会**随着竞争的激烈而逐渐升级**。注意**锁可以升级不可降级**，这种策略是为了**提高获得锁和释放锁的效率**。

### 4.2 锁的升级

JDK1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

#### 4.2.1 偏向锁（只有一个线程进入临界区）

大部分情况下，锁不仅仅不存在多线程竞争， 而是总是由同一个线程多次获得，为了让线程获取锁的代价更低就引入了偏向锁的概念，适用于只有**一个线程访问同步块**的场景。

偏向锁是默认开启的，而且开始时间一般是比应用程序启动慢几秒，如果不想有这个延迟，那么可以设置

**`-XX:BiasedLockingStartUpDelay=0`**；如果不想要偏向锁，那么可以通过**`-XX:-UseBiasedLocking = false`**来设置，关闭后会直接进入轻量级锁。

**偏向锁的加锁：**

1. 检查Mark Word是否为**可偏向状态**，即MarkWord中锁标志是否为‘01’，是否偏向锁是否为‘1’
2. 如果是可偏向锁，则检查Mark Word储存的线程ID是否为当前线程ID，如果是则执行同步代码，否则执行步骤3
3. 如果检查到Mark Word储存的线程ID不是当前线程的ID，则通过CAS操作去修改为本线程的ID，如果修改成功则执行把步骤5，否则执行步骤4
4. 当拥有该锁的线程到达安全点之后，挂起这个线程，升级为轻量级锁
5. 执行同步代码

**偏向锁的撤销：**

偏向锁使用了一种**等到竞争出现才释放锁**的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。 偏向锁的撤销需要等到**全局安全点(在这个时间点上没有正在执行的字节码)**

1. 暂停拥有偏向锁的线程，检查持有偏向锁的线程是否活着，如果不处于活动状态，则将对象头设置为无锁状态，否则设置为被锁定状态。
2. 如果锁对象处于无锁状态，则恢复到无锁状态(01)，以允许其他线程竞争，如果锁对象处于锁定状态，则挂起持有偏向锁的线程，并将对象头Mark Word的锁记录指针改成当前线程的锁记录，锁升级为轻量级锁状态(00)。

#### 4.2.2 轻量级锁（多个线程交替进入临界区）

轻量级锁考虑的是竞争锁对象的**线程不多**，而且线程持有锁的时间也不长的情景。因为阻塞线程需要CPU从用户态转到内核态，代价较大，如果刚刚阻塞不久这个锁就被释放了，那这个代价就有点得不偿失了，因此这个时候就干脆不阻塞这个线程，让它自旋着等待锁释放。

**轻量级锁的加锁：**

1. 线程在执行同步块之前, JVM会先在当前线程的**栈帧中创建用户存储锁记录的空间(Lock Record)**, 并将对象头中的**`MarkWord`**复制到锁记录中
2. 线程尝试使用CAS将对象头中的MarkWord替换为指向锁记录的指针。如果成功，执行步骤3，否则步骤4
3. 更新成功，则当前线程持有该对象锁，并且对象MarkWord锁标志设置为'00'，即表示此对象处于轻量级锁状态
4. 更新失败，jvm先检查对象MarkWord是否指向当前线程栈帧中的锁记录，如果是则执行步骤5，否则执行步骤6
5. 表示锁重入；然后当前线程栈帧中增加一个锁记录（Displaced Mark Word）为null，并指向Mark Word的锁对象，起到一个重入计数器的作用
6. 表示该锁对象已经被其他线程抢占，则进行自旋等待（默认10次），等待次数达到阈值仍未获取到锁，则升级为重量级锁

> 一般线程持有锁的时间都不是太长，仅仅为了这一点时间去挂起线程/恢复线程是得不偿失的，所以，为了让一个线程等待，我们只需要让线程执行一个**忙循环（自旋）**，这项技术就叫做自旋。
>
> 自旋锁是默认开启的，可通过**`--XX:+UseSpinning`**来设置，自旋次数的默认值是10次，用户可以修改**`--XX:PreBlockSpin`**来更改。
>
> **适应性自旋**(自适应的自旋锁)带来的改进就是：自旋的时间不在固定了，而是根据前一次同一个锁上的自旋时间以及锁的拥有者的状态来决定。

**轻量级锁的解锁：**

轻量级锁解锁时, 会使用原子的CAS操作将当前线程的锁记录替换回到对象头, 如果成功, 表示没有竞争发生; 如果失败, 表示当前锁存在竞争, 锁就会升级成重量级锁。

1. 使用CAS操作将当前线程的锁记录替换回到对象头，如果替换成功，则执行步骤2，否则执行步骤3。
2. 如果替换成功，整个同步过程就完成了，恢复到无锁的状态(01)。
3. 如果替换失败，说明有其他线程尝试获取该锁(此时锁已膨胀)，那就要在释放锁的同时，唤醒被挂起的线程

#### 4.2.3 重量级锁（多个线程同时进入临界区）

当有多个锁竞争轻量级锁则会升级为重量级锁，重量级锁正常会进入一个cxq的队列，在调用wait方法之后，则会进入一个waitSet的队列park等待，而当调用notify方法唤醒之后，则有可能进入EntryList

**重量级锁的加锁**

1. 分配一个ObjectMonitor对象，把Mark Word锁标志置为‘10’，然后Mark Word存储指向ObjectMonitor对象的指针。ObjectMonitor对象有两个队列和一个指针，每个需要获取锁的线程都包装成ObjectWaiter对象
2. 多个线程同时执行同一段同步代码时，ObjectWaiter先进入EntryList队列，当某个线程获取到对象的monitor以后进入Owner区域，并把monitor中的owner变量设置为当前线程，同时monitor中的计数器count+1

### 4.3 锁粗化

锁粗化就是将多次连接在一起的加锁、解锁操作合并为一次操作。将多个联系的锁扩展为一个范围更大的锁。例如：

```java
for(int i=0;i<size;i++){
    synchronized(lock){
    }
}
```

锁粗化后：

```java
synchronized(lock){
    for(int i=0;i<size;i++){
    }
}
```

### 4.4 锁消除

锁粗化指的就是虚拟机即使编译器在运行时，如果检测到那些共享数据不可能存在竞争(堆上的数据不会逃逸出当前线程，则认为是线程安全的)，那么就执行锁消除。锁消除可以节省毫无意义的请求锁的时间。

> 前提是java必须运行在server模式（server模式会比client模式作更多的优化），同时必须开启逃逸分析:
>
> -server -XX:+DoEscapeAnalysis -XX:+EliminateLocks
>
> 其中+DoEscapeAnalysis表示开启逃逸分析，+EliminateLocks表示锁消除

```java
public class demo{
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer();
        sb.append("a");
        sb.append("b");
        sb.append("c");
    }
}
```

我们都知道StringBuffer线程安全的，它的append()方法是同步方法。但上述代码中，sb是一个局部变量，它引用的StringBuffer对象是不会被其它线程引用的。所以，这里多次调用append()同步方法时认为是线程安全的，无需加锁。

