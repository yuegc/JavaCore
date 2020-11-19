## 1. Atomic 原子类介绍

Atomic 翻译成中文是原子的意思。在化学上，我们知道原子是构成一般物质的最小单位，在化学反应中是不可分割的。在我们这里 Atomic 是指一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。

所以，所谓原子类说简单点就是具有原子/原子操作特征的类。

根据操作的数据类型，可以将JUC包中的原子类分为4类

**基本类型** 

使用原子的方式更新基本类型

- AtomicInteger：整型原子类
- AtomicLong：长整型原子类
- AtomicBoolean ：布尔型原子类

**数组类型**

使用原子的方式更新数组里的某个元素


- AtomicIntegerArray：整型数组原子类
- AtomicLongArray：长整型数组原子类
- AtomicReferenceArray ：引用类型数组原子类

**引用类型**

- AtomicReference：引用类型原子类

- AtomicMarkableReference：原子更新带有标记的引用类型。该类将 boolean 标记与引用关联起来，可以降低使用 CAS 进行原子更新时可能出现的 ABA 问题的概率，但是不能完全解决。

- AtomicStampedReference ：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

  ```java
   /**
  AtomicMarkableReference是将一个boolean值作是否有更改的标记，本质就是它的版本号只有两个，true和false，
  修改的时候在这两个版本号之间来回切换，这样做并不能解决ABA的问题，只是会降低ABA问题发生的几率而已
  */
  public class SolveABAByAtomicMarkableReference {   
  private static AtomicMarkableReference atomicMarkableReference = new AtomicMarkableReference(100, false);
  
      public static void main(String[] args) {
          Thread refT1 = new Thread(() -> {
              try {
                  TimeUnit.SECONDS.sleep(1);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              atomicMarkableReference.compareAndSet(100, 101, atomicMarkableReference.isMarked(), !atomicMarkableReference.isMarked());
              atomicMarkableReference.compareAndSet(101, 100, atomicMarkableReference.isMarked(), !atomicMarkableReference.isMarked());
          });
  
          Thread refT2 = new Thread(() -> {
              boolean marked = atomicMarkableReference.isMarked();
              try {
                  TimeUnit.SECONDS.sleep(2);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              boolean c3 = atomicMarkableReference.compareAndSet(100, 101, marked, !marked);
              System.out.println(c3); // 返回true,实际应该返回false
          });
  
          refT1.start();
          refT2.start();
      }
  }
  ```

**对象的属性修改类型**

- AtomicIntegerFieldUpdater:原子更新整型字段的更新器
- AtomicLongFieldUpdater：原子更新长整型字段的更新器
- AtomicReferenceFieldUpdater：原子更新引用类型里的字段

**CAS ABA 问题**

- 描述: 第一个线程取到了变量 x 的值 A，然后巴拉巴拉干别的事，总之就是只拿到了变量 x 的值 A。这段时间内第二个线程也取到了变量 x 的值 A，然后把变量 x 的值改为 B，然后巴拉巴拉干别的事，最后又把变量 x 的值变为 A （相当于还原了）。在这之后第一个线程终于进行了变量 x 的操作，但是此时变量 x 的值还是 A，所以 compareAndSet 操作是成功。

  ```java
  public class AtomicIntegerDefectDemo {
      public static void main(String[] args) {
          defectOfABA();
      }
  
      static void defectOfABA() {
          final AtomicInteger atomicInteger = new AtomicInteger(1);
  
          Thread coreThread = new Thread(
                  () -> {
                      final int currentValue = atomicInteger.get();
                      System.out.println(Thread.currentThread().getName() + " ------ currentValue=" + currentValue);
  
                      // 这段目的：模拟处理其他业务花费的时间
                      try {
                          Thread.sleep(300);
                      } catch (InterruptedException e) {
                          e.printStackTrace();
                      }
  
                      boolean casResult = atomicInteger.compareAndSet(1, 2);
                      System.out.println(Thread.currentThread().getName()
                              + " ------ currentValue=" + currentValue
                              + ", finalValue=" + atomicInteger.get()
                              + ", compareAndSet Result=" + casResult);
                  }
          );
          coreThread.start();
  
          // 这段目的：为了让 coreThread 线程先跑起来
          try {
              Thread.sleep(100);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
  
          Thread amateurThread = new Thread(
                  () -> {
                      int currentValue = atomicInteger.get();
                      boolean casResult = atomicInteger.compareAndSet(1, 2);
                      System.out.println(Thread.currentThread().getName()
                              + " ------ currentValue=" + currentValue
                              + ", finalValue=" + atomicInteger.get()
                              + ", compareAndSet Result=" + casResult);
  
                      currentValue = atomicInteger.get();
                      casResult = atomicInteger.compareAndSet(2, 1);
                      System.out.println(Thread.currentThread().getName()
                              + " ------ currentValue=" + currentValue
                              + ", finalValue=" + atomicInteger.get()
                              + ", compareAndSet Result=" + casResult);
                  }
          );
          amateurThread.start();
      }
  }
  ```

  **输出结果**

  ```
  Thread-0 ------ currentValue=1
  Thread-1 ------ currentValue=1, finalValue=2, compareAndSet Result=true
  Thread-1 ------ currentValue=2, finalValue=1, compareAndSet Result=true
  Thread-0 ------ currentValue=1, finalValue=2, compareAndSet Result=true
  ```

## 2. 基本原子类型

#### 2.1 基本类型原子类介绍

使用原子的方式更新基本类型

- AtomicInteger：整型原子类
- AtomicLong：长整型原子类
- AtomicBoolean ：布尔型原子类

上面三个类提供的方法几乎相同，这里以 AtomicInteger 为例子来介绍。

 **AtomicInteger 类常用方法**

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

### 2.2 AtomicInteger 常见方法使用

```java
public class AtomicIntegerTest {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		int temvalue = 0;
		AtomicInteger i = new AtomicInteger(0);
		temvalue = i.getAndSet(3);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:0;  i:3
		temvalue = i.getAndIncrement();
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:3;  i:4
		temvalue = i.getAndAdd(5);
		System.out.println("temvalue:" + temvalue + ";  i:" + i);//temvalue:4;  i:9
	}
}
```

### 2.3 基本数据类型原子类的优势

**①多线程环境不使用原子类保证线程安全（基本数据类型）**

```java
class Test {
        private volatile int count = 0;
        //若要线程安全执行执行count++，需要加锁
        public synchronized void increment() {
                  count++; 
        }

        public int getCount() {
                  return count;
        }
}
```

**②多线程环境使用原子类保证线程安全（基本数据类型）**

```java
class Test2 {
        private AtomicInteger count = new AtomicInteger();

        public void increment() {
                  count.incrementAndGet();
        }
      //使用AtomicInteger之后，不需要加锁，也可以实现线程安全。
       public int getCount() {
                return count.get();
        }
}
```

### 2.4 AtomicInteger 线程安全原理简单分析

部分源码：

```java
// setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
          	// objectFieldOffset为本地方法
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

