### J.U.C



<img src="http://img.miilnvo.xyz/ofjz1.jpg" alt="ofjz1" style="zoom:50%;" />



---

#### Java内存模型

![ihlsq](http://img.miilnvo.xyz/ihlsq.png)

每条线程都有自己独立的工作内存，里面保存了主内存的变量副本

线程间的变量副本的传递需要通过主内存完成，而工作内存把最新的值刷新到主内存的时间是无法确定的

Synchronized和Lock会确保在锁释放之前，将对变量副本的修改值刷新到主内存当中



---

#### volatile

适用于表示状态的变量

* 原子性：无法保证

  ```java
  volatile int num = 0;
  num++;  // 此步可分成三步，多条线程可能会同时读取变量num，所以对变量num的写入操作不能依赖当前的值
  num = a + 1;  // 因为不存在对变量num的读取，所以多条线程同时执行不会产生其它波动值
  ```
  
  > 工作内存在更新变量副本后并不会刷新其它工作内存的值
  
* 可见性：对volatile变量进行写操作时，会强制将最新值刷新到主内存中；进行读操作时，会强制从主内存中读取

  > 本质是加上了LOCK指令

  >  use必须和load-read连续，assign必须和store-write连续

  > 保证了只要A线程修改后，B线程再进行读取时就可以读到最新值

* 有序性：不会把volatile变量前的语句放到其后执行，反之亦然

【参考】

<https://blog.csdn.net/guyuealian/article/details/52525724>

https://blog.csdn.net/weililansehudiefei/article/details/70904111



#### CAS

Compare And Swap（比较和交换）

Unsafe#`compareAndSwapInt()`方法属于native方法，本质是执行了加上LOCK前缀（保证多核情况下的原子性）的`cmpxchg`指令（弥补了volatile缺失的原子性），该指令的作用是比较主内存中的值是否与工作内存的值相同，如果相同则用新值替换主存中的旧值（并不包括`for(;;)`循环）

> 并发：单核CPU处理多线程，即多个事件间隔发生
>
> 并行：多核CPU处理多线程，即多个事件同时发生
>
> 在单核CPU、没有阻塞的程序中使用多线程是没有必要的，多线程要在多核CPU上才能发挥优势

> 单核CPU中，能够在一个指令中完成的操作都可以看作为原子操作，因为中断只发生在指令间，所以在单核CPU中执行`cmpxchg`指令不需要加LOCK。而在多核CPU中，可能同一时间还有其它线程会访问内存，所以需要用LOCK锁总线

缺点：
1. ABA问题（无法获知初次读取和第二次读取时中间的变化）

   > 解决方法：添加版本号或使用AtomicStampedReference

2. 只能保证一个共享变量的原子操作

3. 若循环时间长，则CPU开销大

【参考】

<https://www.jianshu.com/p/ae25eb3cfb5d>

<https://www.jianshu.com/p/da705f9fd7cd>



---

#### 阻塞/唤醒

LockSupport类

不需要像Object#`wait()` / `notify()`放在同步代码块中使用

`unpark() `的实现是使_counter值为1，所以可以在`park()`之前调用，但多次无效

Synchronzed使线程进入到BLOCKED状态，而LockSupprt#`park()`方法使线程进入到WAITING状态

```java
public class LockSupport {
    private static final sun.misc.Unsafe UNSAFE;
    private static final long parkBlockerOffset;  // 对应Thread的parkBlocker属性
    
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);  // 用于记录线程是被谁阻塞的
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
    
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
}
```

【参考】

https://www.cnblogs.com/qingquanzi/p/8228422.html



#### 非阻塞

```java
for (;;) {
    int current = get();
    int next = current + 1;
    if (cas(current, next)) {
        return next;
    }
}
```

```java
// Unsafe类
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
```



#### AQS

AbstractQueuedSynchronizer（抽象队列同步器）

![5yh7u](http://img.miilnvo.xyz/5yh7u.png)

> 同步队列中的等待线程是等待获取锁，等待队列中的等待线程是等待唤醒

##### 数据结构

* AbstractOwnableSynchronizer抽象类

  ```java
  public abstract class AbstractOwnableSynchronizer {
      private transient Thread exclusiveOwnerThread;  // 在独占模式下获取当前同步锁的线程
  }
  ```

* AbstractQueuedSynchronizer抽象类

  ```java
  public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
      private static final long XXXOffset;    // 各种属性的偏移值，用于CAS操作
      private transient volatile Node head;   // 同步队列的头节点
      private transient volatile Node tail;   // 同步队列的尾节点
      private volatile int state;             // 子类通过此属性来判断是否能获取到锁！
      
      // 双向链表结构
      static final class Node {
          // 1   CANCELLED  等到超时或已被中断
          // 0              默认
          // -1  SIGNAL     准备park，可以被唤醒
          // -2  CONDITION  处于等待队列	
          // -3  PROPAGATE  与共享模式有关？
          volatile int waitStatus;
          volatile Node prev;
          volatile Node next;
          volatile Thread thread;  // 当前节点所对应的线程
          Node nextWaiter;  // 专门用于ConditionObject中的节点，结构为单向队列
      }
      
      // 包含阻塞、唤醒等操作，类似Object#wait()/notify()
      public class ConditionObject implements Condition {
          private transient Node firstWaiter;  // 等待队列的头节点
          private transient Node lastWaiter;   // 等待队列的尾节点 
          
          public final void await() {...}  // 当前线程进行park，会响应中断
          public final void awaitUninterruptibly() {...}  // 当前线程park，不会响应中断
          public final void signal() {...}  // 某个线程取消等待
          public final void signalAll() {...}  // 所有线程取消等待
      } 
  }
  ```

##### 同步队列

独占：只有一个线程能执行资源（ReentrantLock）

共享：多个线程可以同时执行资源（CountDownLatch、Semaphore）


* 独占方式获取

  ```java
  public final void acquire(int arg) {
      if (!tryAcquire(arg) &&
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          selfInterrupt();
  }
  ```

  `tryAcquire()`：由具体子类实现，可以通过判断state属性的值来确定是否可以获取锁

  > 此方法之所以没有定义成abstract，是因为在独占模式下只用实现tryAcquire和tryRelease，而共享模式下只用实现tryAcquireShared和tryReleaseShared。如果定义成abstract，那么每个模式也要去实现另一模式下的接口

  `addWaiter()`：把当前线程包装为Node对象，并加入到同步队列的尾部

  `acquireQueued()`：在循环中尝试获取锁，如果失败则线程进入park状态

  `selfInterrupt()`：因为不会响应中断操作，所以只有在获取到锁后才执行中断

* 共享方式获取

  ```java
  public final void acquireShared(int arg) {
      if (tryAcquireShared(arg) < 0)
          doAcquireShared(arg);
  }
  ```

  `tryAcquireShared()`：由具体子类实现

  `doAcquireShared()`：与独占方式的后三个方法逻辑相同

  > tryAcquire()的返回值是boolean，tryAcquireShared()的返回值是int，这是因为在共享方式下可以有多个线程获取锁，所以需要返回一个剩余可获得值

* 独占方式释放

  ```java
  public final boolean release(int arg) {
      if (tryRelease(arg)) {
          Node h = head;
          if (h != null && h.waitStatus != 0)
              unparkSuccessor(h);
          return true;
      }
      return false;
  }
  ```

  `tryRelease()`： 由具体子类实现，可以直接把state属性的值-1

  > 解锁时会判断当前线程是否为加锁时所记录的线程

  `unparkSuccessor()`：从同步队列的尾部往前判断，找到最后一个可以unpark的线程

* 共享方式释放

  ```java
  public final boolean releaseShared(int arg) {
      if (tryReleaseShared(arg)) {
          doReleaseShared();
          return true;
      }
      return false;
  }
  ```

​	`tryReleaseShared()`：由具体子类实现

​	`doReleaseShared()`：通过判断头节点是否变化来循环唤醒等待中的其它共享节点（尝试唤醒多个）

##### 等待队列

* 阻塞线程

  ```java
  public final void await() throws InterruptedException {
      if (Thread.interrupted())
          throw new InterruptedException();
      Node node = addConditionWaiter();  // 创建当前线程的节点，添加到等待队列
      int savedState = fullyRelease(node);  // 释放锁，同时唤醒同步队列的下一个节点
      int interruptMode = 0;
      // 判断当前节点是否已经移动到了同步队列中，即是否在其它地方调用了signal()/signalAll()方法
      while (!isOnSyncQueue(node)) {
          LockSupport.park(this);  // park当前节点
          if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
              break;
      }
      if (acquireQueued(node, savedState) && interruptMode != THROW_IE)  // 重新获取锁
          interruptMode = REINTERRUPT;
      if (node.nextWaiter != null)
          unlinkCancelledWaiters();
      if (interruptMode != 0)
          reportInterruptAfterWait(interruptMode);
  }
  ```

  ![60pyh](http://img.miilnvo.xyz/60pyh.jpg)

* 唤醒线程

  ```java
  private void doSignal(Node first) {
      do {
          if ((firstWaiter = first.nextWaiter) == null)
              lastWaiter = null;
          first.nextWaiter = null;
      } while (!transferForSignal(first)  // 把等待队列的节点移动到同步队列，并unpark该节点
               && (first = firstWaiter) != null);
  }
  
  private void doSignalAll(Node first) {
      lastWaiter = firstWaiter = null;
      do {
          Node next = first.nextWaiter;
          first.nextWaiter = null;
          transferForSignal(first);  // 把等待队列的节点移动到同步队列，并unpark该节点
          first = next;
      } while (first != null);
  }
  ```

  ![l7wnm](http://img.miilnvo.xyz/l7wnm.jpg)

> 无论是释放锁之后的唤醒，还是signal()方法的唤醒，最终都是在同步队列中进行的，并且是按照队列中的先后顺序，会先唤醒头结点

【参考】

<https://www.jianshu.com/p/da9d051dcc3d>

<https://www.jianshu.com/p/28387056eeb4>



---

#### 锁

乐观锁：假设不会有资源竞争，如果失败了就循环重试（循环 + CAS操作）

悲观锁：假设一定会有资源竞争，在操作前先加锁（synchronized）

可重入锁：该锁可以被单个线程多次获取（synchronized、ReentrantLock）

公平锁：其它线程依次排队获取锁（ReentrantLock.FairSync）

非公平锁：其它线程随机获取锁（ReentrantLock.NonfairSync）

独占锁： 该锁每次只能被一个线程持有（synchronized、ReentrantLock、ReentrantReadWriteLock.WriteLock）

共享锁：该锁可以被多个线程持有（ReentrantReadWriteLock.ReadLock）

> 循环 + CAS操作 = 乐观锁 ，不会阻塞线程。ReentrantLock有CAS操作，在循环内会阻塞线程，不是乐观锁

* Synchronized

  * 实现方式

    * Class文件层面

      修饰同步代码块：monitorenter和monitorexit字节码指令

      修饰普通/静态方法：方法修饰符上的ACC_SYNCHRONIZED标志

    * Native方法层面

      Java对象头中的第一部分Mark Word包括了锁状态等信息

      C++的ObjectMonitor对象为monitor机制提供了支持，包含\_WaitSet和\_EntryList两个队列，用来管理线程的阻塞和唤醒
      
      > ObjectMonitor ～ AQS
      >
      > \_EntryList ～ 同步队列
      >
      > \_WaitSet   ～ 等待队列
      >
      > ObjectWaiter ～ Node

  * 锁升级（v1.6）

    偏向锁 ==> [锁膨胀、锁撤销] ==> 轻量级锁 ==> [自旋锁、自适应自旋锁] ==> 重量级锁

    * 偏向锁：偏向于第一个获取它的线程，之后不需要再进行加锁或解锁

      > 用于同一线程反复获得同一锁的情况

    * 轻量级锁：当锁已经被其它线程获取后，会一直循环重试，直到获取锁（自旋锁）
    
      > 用于短时间持有锁的情况
    
    * 重量级锁：当锁已经被其它线程获取后，阻塞当前线程，进行休眠
    
      > 用于很多线程同时竞争的情况

  【参考】

  https://juejin.im/post/6844904069845221384#heading-3

  <https://blog.csdn.net/tongdanping/article/details/79647337>

* ReentrantLock

    加锁：

    1. `compareAndSetState()`：非公平锁会先尝试直接获取锁，无需排队处理，若成功则直接返回true
    2. `tryAcquire()`：重写AQS的`tryAcquire()`方法，判断state状态，若为0则获取成功，返回true
    3. 执行AQS以独占方式获取锁的后三步

    解锁：

    1. `tryRelease()`：重写AQS的`tryRelease()`方法，每次执行都会减少state值
    2. `unparkSuccessor()`：当state为0时唤醒头节点之后的那条线程

    ```java
    public interface Lock {
        void lock();               // 获取锁
        void lockInterruptibly();  // 获取可中断锁
        boolean tryLock();         // 尝试获取一次锁，失败返回false
        boolean tryLock(long time, TimeUnit unit);
        void unlock();             // 释放锁
        Condition newCondition();  // 包含await()、signal()、signalAll()等方法
    }
    ```

    ```java
    public class ReentrantLock implements Lock {
        
        private final Sync sync;
        
        public ReentrantLock() {
            sync = new NonfairSync();  // 默认是非公平锁
        }
        
        abstract static class Sync extends AbstractQueuedSynchronizer {...}
        // 非公平锁
        static final class NonfairSync extends Sync {
            final void lock() {
                if (compareAndSetState(0, 1))
                    setExclusiveOwnerThread(Thread.currentThread());
                else
                    acquire(1);
            }
        }
        // 公平锁
        static final class FairSync extends Sync {
            final void lock() {
                acquire(1);
            }
        }
    }
    ```

    ```java
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 表示没有其它线程获取锁，此线程可以进行加锁
        if (c == 0) {
            if (!hasQueuedPredecessors() &&  // NonfairSync没有这一步判断
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 可重入锁的逻辑，对state+1即可
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 获取锁失败
        return false;
    }
    ```

    NonfairSync类和FairSync类中的`tryAcquire()`都是通过对AQS的state进行简单的加减来判断是否能获取到锁，唯一的区别是FairSync类的`tryAcquire()`多执行了一步`hasQueuedPredecessors()`，此方法限制了只有在队头（实际为第二个）的线程才能获取到锁

* ReentrantReadWriteLock

  当有线程获取读锁时，不允许其它线程获得写锁，但可以获取读锁

  当有线程获得写锁时，不允许其它线程获得读锁和写锁

  > 读读不互斥，读写互斥，写写互斥

  > state属性的低16位表示读锁的获取次数，高16位表示写锁的获取次数
  
  ```java
  public interface ReadWriteLock {
      Lock readLock();
      Lock writeLock();
  }
  
  public class ReentrantReadWriteLock implements ReadWriteLock {
      private final ReentrantReadWriteLock.ReadLock readerLock;
      private final ReentrantReadWriteLock.WriteLock writerLock;
  	final Sync sync;
      
      public ReentrantReadWriteLock.ReadLock readLock() { return readerLock; }
      public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
  
      // 读锁和写锁使用同一个AQS
      public ReentrantReadWriteLock(boolean fair) {
          sync = fair ? new FairSync() : new NonfairSync();
          readerLock = new ReadLock(this);
          writerLock = new WriteLock(this);
      }
      
      public static class ReadLock implements Lock {
          public void lock() {
              sync.acquireShared(1);  // AQS共享方式获取
          }
      }
      public static class WriteLock implements Lock {
          public void lock() {
              sync.acquire(1);  // AQS独占方式获取
          }
      }
      
      abstract static class Sync extends AbstractQueuedSynchronizer {...}
  }
  ```
  
  【参考】
  
  <https://www.jianshu.com/p/4a624281235e>
  
* LockSupport

  （略）



#### 同步工具

* CountDownLatch类

  A线程需要等B、C、D线程都执行完成后才能执行

  ```java
  CountDownLatch latch = new CountDownLatch(N);  // AQS的state=N
  for (int i = 0; i < N; i++) {
      pool.execute(() -> {
          // ...
          // 调用AQS的releaseShared()，每条线程都会使state-1
          // 当state=0时才算释放成功，会唤醒之前所有被await()方法阻塞的线程
          latch.countDown();  
          // ...
      });
  }
  // 调用AQS的acquireSharedInterruptibly()，state!=0时当前线程会阻塞
  // 使用共享锁的原因是await()方法可以在多个地方被调用
  latch.await();  
  // 继续执行A线程...
  ```

* CyclicBarrier类

  A、B、C、D线程相互等待，直到达到某个状态时再同时执行

  ```java
  CyclicBarrier barrier = new CyclicBarrier(N, () -> {...});  // 所有线程达到屏障点后执行的任务
  for (int i = 1; i < N; i++) {
      pool.execute(() -> {
          // ...
          barrier.await();
          // 等待所有线程达到屏障点后才会继续执行...
      });
  }
  ```

  ```java
  public class CyclicBarrier {
      private final ReentrantLock lock = new ReentrantLock();
      private final Condition trip = lock.newCondition();
      private final int parties;  // 线程总数
      private final Runnable barrierCommand;  // 构造函数中的Lambda
      private Generation generation = new Generation();  // 每一个Cyclic周期代表一个年代
      private int count;  // 当前还未到达屏障点的线程
      
      private int dowait(boolean timed, long nanos) {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              final Generation g = generation;
              // ...
              int index = --count;
              if (index == 0) {  // 表示最后一个线程到达屏障点
                  final Runnable command = barrierCommand;
                  if (command != null)
                      command.run();
                  nextGeneration();  // 更新状态！
                  return 0;
              }
              for (;;) {
                  if (!timed)
                      trip.await();  // 新建节点，并添加到等待队列中，park当前线程，释放Lock锁
                  // ...
                  if (g != generation)  // 在nextGeneration()中会创建新的年代
                      return index;
              }
          } finally {
              // 最后一个线程：对应上面的lock.lock()，会传播唤醒同步队列中的节点
              // 其它被唤醒的线程：对应被唤醒后的acquireQueued()，会传播唤醒同步队列中的节点
              lock.unlock();
          }
      }
      
      private void nextGeneration() {
          trip.signalAll();  // 把等待队列的节点全部移动到同步队列
          count = parties;   // 初始化线程数量值
          generation = new Generation();  // 创建新的年代
      }
  }
  ```

* Semaphore类

  AQS共享模式的应用

  ```java
  public class Semaphore {
      Semaphore semaphore = new Semaphore(N);  // state=N
      for (int i = 1; i < N; i++) {
          pool.execute(() -> {
              semaphore.acquire();  // 调用AQS的acquireSharedInterruptibly()方法
              // ...
              semaphore.release();  // 调用AQS的releaseShared()方法
          });
      }
  }
  ```

  > 当N>1时为共享锁，当N=1时为独占锁

  > 与读锁的区别：release()方法可以在acquire()方法前执行



#### 并发集合

* ConcurrentHashMap类

  v1.7：分段锁

  v1.8：数组 + 链表 / 红黑树 + CAS + Synchronized

  ```java
  final V putVal(K key, V value, boolean onlyIfAbsent) {
      // ...
      int hash = spread(key.hashCode());
      for (Node<K,V>[] tab = table;;) {
          // 1.初始化table
          if (tab == null || (n = tab.length) == 0)
              tab = initTable();
          // 2.当Key的hash值对应的数组下标为null时
          // 包含Unsafe.getObjectVolatile()方法
          else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
              // 用CAS操作把Node添加数
              if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                  break;
          // 3.当数组不为null时
          } else {
              synchronized (f) {  // 只锁对应数组里的Node(首节点)
                  // ...
              }
          }
      }
      // ...
  }
  ```
  

扩容条件：当前总个数超过阈值sizeCtl（0.75扩容因子） 或 链表的个数超过8个且数组总长度小于64 ==> 原来的两倍大小

链表转红黑树条件：链表的个数超过8个且数组总长度大于64

> Q：怎么使HashMap变成线程安全（即HashMap和ConcurrentHashMap的区别）
>
> A：1.table加上volatile（但是对数组元素仍无可见性）
>
>​      2.查询数组元素时要使用getObjectVolatile()方法
>
>​      3.修改时使用CAS操作或对首节点加锁

【参考】

https://blog.csdn.net/tp7309/article/details/76532366

* ConcurrentSkipListMap类

  基于跳表，空间换时间

  Key值有序

* BlockingQueue接口（阻塞队列）

  ![m82re](http://img.miilnvo.xyz/m82re.png)

  | 所在类 \| 结果 | Collection \| 抛异常 | Queue \| 返回boolean或null | BlockingQueue \| 阻塞 |
| :------------: | :------------------: | :------------------------: | :-------------------: |
  |      入队      |       `add(e)`       |         `offer(e)`         |       `put(e)`        |
  |      出队      |      `remove()`      |          `poll()`          |       `take()`        |
  |      获取      |     `element()`      |          `peek()`          |           /           |
  
  * ArrayBlockingQueue类

    ```java
    public class ArrayBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, Serializable {
        final Object[] items;  // 数组结构，即有界
        int count;		// 总数
        int putIndex;	// 插入时的下标
        int takeIndex;	// 移除时的下标
        final ReentrantLock lock;  // 读写共用一套锁
        private final Condition notEmpty;	// 当队列空时，读线程会在此等待
        private final Condition notFull;	// 当队列满时，写线程会在此等待
    }
    ```
  
  * LinkedBlockingQueue类

    ```java
    public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
        private final AtomicInteger count = new AtomicInteger();  // 总数
        transient Node<E> head;
        private transient Node<E> last;
        // 读写各用一套锁，在高并发场景下性能比前者更好
        private final ReentrantLock takeLock = new ReentrantLock();
        private final Condition notEmpty = takeLock.newCondition();
        private final ReentrantLock putLock = new ReentrantLock();
        private final Condition notFull = putLock.newCondition();
    	
        // 链表结构，即无界（最大容量为Integer.MAX_VALUE）
        static class Node<E> {
            E item;
            Node<E> next;
            Node(E x) { item = x; }
        }
    }
    ```
  
  * PriorityBlockingQueue类：数组 + 二叉堆排序 + 自动扩容的阻塞队列

    `put()`方法不会阻塞

    节点需要实现Comparable接口

    保证每次出队都是优先级最高的元素


* ConcurrentLinkedQueue类（非阻塞队列）

   head/tail并非总是指向队列的头/尾节点，所以允许队列处于不一致的状态

  ```java
  public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
  implements Queue<E>, java.io.Serializable {
	    private transient volatile Node<E> head;  // 头结点
	    private transient volatile Node<E> tail;  // 尾节点
  	
	    // 链表结构
	    private static class Node<E> {
          volatile E item;
          volatile Node<E> next;
	    }
  	
	    // 添加元素
	    public boolean offer(E e) {
          checkNotNull(e);
          final Node<E> newNode = new Node<E>(e);
          for (Node<E> t = tail, p = t;;) {
              Node<E> q = p.next;
              if (q == null) {
                  if (p.casNext(null, newNode)) {
                      if (p != t)
                          casTail(t, newNode);
                      return true;
                  }
              }
              else if (p == q)
                  p = (t != (t = tail)) ? t : head;
              else
                  p = (p != t && t != (t = tail)) ? t : q;
          }
      }
  }
	```
  
  【参考】
  
  <https://www.ibm.com/developerworks/cn/java/j-lo-concurrent/>

* CopyOnWriteArrayList类

  写时复制：添加元素时不直接往当前容器添加，而是复制出一个新的容器，然后往新的容器里添加元素，最后将原容器的引用指向新的容器

  适用于读多写少的并发场景

  > 读读不互斥，读写不互斥（并非一个容器），写写互斥

  ```java
  public class CopyOnWriteArrayList<E> {
      final transient ReentrantLock lock = new ReentrantLock();
      private transient volatile Object[] array;
      // 写时加锁，避免在多线程环境下复制出多个容器
      public boolean add(E e) {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              Object[] elements = getArray();
              int len = elements.length;
              Object[] newElements = Arrays.copyOf(elements, len + 1);  // 复制出一个副本
              newElements[len] = e;   // 在此副本上进行修改
              setArray(newElements);  // 将引用指向此副本
              return true;
          } finally {
              lock.unlock();
          }
      }
      // 读时不加锁，如果此时正在添加新元素，那么会读到旧数据（只能保证最终一致性，不能保证实时一致性）
      public E get(int index) {
          Object[] a = getArray();
          return (E) a[index];
      }
  }
  ```

  > ArrayList — CopyOnWriteArrayList
  >
  > HashSet  — CopyOnWriteArraySet（基于CopyOnWriteArrayList）



#### 多线程

##### Future

<img src="http://img.miilnvo.xyz/n5poq.jpg" alt="n5poq" style="zoom: 50%;" />

* Callable接口：与Runnable接口的作用相似，只有一个`call()`方法，有返回值且可以拋异常

* Runnable接口：只有一个`run()`方法，无返回值

* Future接口：包含`get()`、`cancel()`、`isCancelled()`、`isDone()`等五个方法，可以获取任务结果、中断任务、判断任务是否完成

    > 表示一个任务的生命周期

* RunnableFuture接口：继承了上面两个接口

* FutureTask类：实现了上面的接口

  ```java
  public class FutureTask<V> implements RunnableFuture<V> {
      private volatile int state;         // 任务生命周期的各个状态
      private Callable<V> callable;       // 在构造函数中赋初始值
      private Object outcome;             // Callable的call()的返回结果
      private volatile Thread runner;     // 当前运行的线程
      private volatile WaitNode waiters;  // 被阻塞的等待队列
      
      public void run() {...}	 // 同步执行Callable的call()方法，然后会唤醒所有阻塞的线程
      public V get() {...}     // 获取结果，若线程还未执行完毕，则会阻塞当前线程，等待唤醒
  }
  ```

  ```java
  public void demo() {
      Callable<String> callable = () -> {
          return "result";
      };
  
      // 方式一
      FutureTask<String> future = new FutureTask<>(callable);
      new Thread(future).start();
      String result = future.get();
      // 方式二
      ExecutorService executor = Executors.newCachedThreadPool();
      Future<String> future = executor.submit(callable);
      String result = future.get();
  }
  ```

  > 使用了适配器模式的两个地方：
  >
  > 1. Runnable诞生于v1.0，而Callable诞生于v1.5。为了能适配新的接口，FutureTask实现了Runnable接口，并拥有一个Callable属性
  >
  > 2. 当FutureTask构造函数的参数使用Runnable时，为了保存到Callable属性使用了Executors#RunnableAdapter()，此内部类实现了Callable接口，并拥有一个Runnable属性

* CompletableFuture类（v1.8）

  使用Future的实现类难以表述多个结果之间的联系

  提供了类似Guava的ListenableFuture监听器的方法
  
  ```java
  public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
      private static final Executor asyncPool = useCommonPool ?
          ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();  // 拥有一个线程池
      
      static final class AsyncRun extends ForkJoinTask<Void>
              implements Runnable, AsynchronousCompletionTask {...}
  
      // 创建，有返回值
      public static <U> CompletableFuture<U> supplyAsync(Supplier supplier) {...}
      // 创建，无返回值
      public static CompletableFuture<Void> runAsync(Runnable runnable) {...}
      
      // 把上个阶段的结果作为入参，有返回值
      public <U> CompletableFuture<U> thenApply(Function fn) {...}
      // 把上个阶段的结果作为入参，无返回值
      public CompletableFuture<Void> thenAccept(Consumer action) {...}
      // 无入参，无返回值
      public CompletableFuture<Void> thenRun(Runnable action) {...}
      
      // 当正常返回结果时执行
      public CompletableFuture<T> whenComplete(BiConsumer action) {...}
      // 当返回异常时执行
      public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn) {...}
      
      // 与另一个任务的结果进行合并处理
      public <U,V> CompletableFuture<V> thenCombine(CompletionStage other, BiFunction fn) {...}
      // 当任意一个任务执行完后再执行
      public static CompletableFuture<Object> anyOf(CompletableFuture... cfs) {...}
      // 当全部任务执行完后再执行
      public static CompletableFuture<Void> allOf(CompletableFuture... cfs) {...}
  }
  ```
  
  > 如果方法名中加上了Async，则表示异步执行
  > 参数中的BiFunction、BiConsumer意味着Lambda表达式有两个入参
  
  > 使用 CompletableFuture future = CompletableFuture.supplyAsync(this::calculate, executorService);
  >
  > 代替 Future future = executorService.submit(this::calculate);
  
  【参考】
  
  <https://www.jianshu.com/p/6bac52527ca4>

##### Executor

<img src="http://img.miilnvo.xyz/ra6a0.jpg" alt="ra6a0" style="zoom:50%;" />

* Executor接口：线程执行的入口

    ```java
    public interface Executor {
        void execute(Runnable command);
    }
    ```

* ExecutorService接口：包含执行、关闭等方法

    ```java
    public interface ExecutorService extends Executor {
        <T> Future<T> submit(Callable<T> task);
        Future<?> submit(Runnable task);
        <T> Future<T> submit(Runnable task, T result);
        void shutdown();
    }
    ```

* AbstractExecutorService抽象类：实现了ExecutorService的部分方法

* ThreadPoolExecutor类：具体实现类

    ```java
    public class ThreadPoolExecutor extends AbstractExecutorService {
        // 高3位表示线程池的运行状态，低29位表示运行的worker数量（约5亿）
        private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
        // 五种线程状态
        private static final int RUNNING    = -1 << COUNT_BITS;
        private static final int SHUTDOWN   =  0 << COUNT_BITS;
        private static final int STOP       =  1 << COUNT_BITS;
        private static final int TIDYING    =  2 << COUNT_BITS;
        private static final int TERMINATED =  3 << COUNT_BITS;
        
        private final ReentrantLock mainLock = new ReentrantLock();
        private final BlockingQueue<Runnable> workQueue;  // 阻塞队列
        private final HashSet<Worker> workers = new HashSet<Worker>();  // 活动线程集合
    
        public ThreadPoolExecutor(
    		int corePoolSize,  		// 创建后不会释放的核心线程数
    		int maximumPoolSize,	// 允许创建的最大线程数
    		long keepAliveTime,  	// 线程空闲的最大时间
    		TimeUnit unit,  		// 时间单位
    		BlockingQueue<Runnable> workQueue,	// 用于保存待执行的任务的阻塞队列
    		ThreadFactory threadFactory,		// 用于创建线程的工厂
    		RejectedExecutionHandler handler)	// 拒绝策略
        {...}
        
        // 自定义的扩展方法
        protected void beforeExecute(Thread t, Runnable r) { }
        protected void afterExecute(Runnable r, Throwable t) { }
        protected void terminated() { }
        
        // 四种拒绝策略
        // 1.丢弃任务，并抛出RejectedExecutionException异常（默认策略）
        public static class AbortPolicy implements RejectedExecutionHandler {...｝
        // 2.丢弃任务
        public static class DiscardPolicy implements RejectedExecutionHandler {...}
        // 3.移除等待队列中的队头任务，重新尝试加入队列
        public static class DiscardOldestPolicy implements RejectedExecutionHandler {...}
        // 4.由主线程直接执行此任务 
        public static class CallerRunsPolicy implements RejectedExecutionHandler {...}
    }
    ```

    * `execute()`

      创建核心线程 ==> 进入等待队列 ==> 创建非核心线程 ==> 执行拒绝策略

      ```java
      public void execute(Runnable command) {
          if (command == null)
              throw new NullPointerException();
          int c = ctl.get();
          // 1.未达到核心线程数，创建核心线程并执行任务
          if (workerCountOf(c) < corePoolSize) { 
              if (addWorker(command, true))  
                  return;
              c = ctl.get();
      	}
          // 2.已达到核心线程数，把任务放入阻塞队列
          if (isRunning(c) && workQueue.offer(command)) {  
              int recheck = ctl.get();
              // 2-1.若此时线程池关闭则移除刚添加的任务并执行拒绝策略
              if (! isRunning(recheck) && remove(command))
                  reject(command);
              // 2-2.若线程总数为零则创建一个空任务的Worker，避免无核心线程数时任务始终放在队列中
              else if (workerCountOf(recheck) == 0)
                  addWorker(null, false);
          }
          // 3.队列已满，创建非核心线程来执行刚添加的任务
          else if (!addWorker(command, false))
              // 4.执行拒绝策略
              reject(command);  
      }
      ```
      * `addWorkder()`

        1. `compareAndIncrementWorkerCount(ctl)`：用循环+CAS操作使ctl值加1

        2. `new Worker(firstTask)`：把任务封装成Worker对象，同时创建了一条新的线程

           ```java
           // Worker继承了AQS，目的是为了使用Worker锁和中断来实现关闭线程池
           // 1.Worker执行任务时加锁
           // 2.Worker空闲时解锁
           private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
               final Thread thread;
               Runnable firstTask;
           }
           ```

        3. `workers.add(w)`：添加到Worker集合

        4. `t.start()`：异步执行Worker的`run()`方法，即`runWorker(this)`

      * Worker#`runWorker()`

        ```java
        final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock();  // ==> setState(0)，之后shutdown()方法就能设置中断标记（多此一举？）
            boolean completedAbruptly = true;
            try {
                // 不断地获取等待队列中的任务！
                while (task != null || (task = getTask()) != null) {
                    w.lock();  // 获取Worker的互斥锁，之后shutdown()方法就不能设置中断标记
                    
                    // 当线程池的状态为STOP时，除了等待队列中的线程会标记中断，当前线程也会标记中断
                    // 由run()方法来决定是否对InterruptedException作出处理来"终止任务"
                    // 之后会在当前任务执行完，在getTask()中处理后退出While循环
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                         (Thread.interrupted() &&
                          runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            task.run();  // 执行任务的真正逻辑
                        } finally {
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        w.completedTasks++;
                        w.unlock();  // 执行完后释放Worker的互斥锁
                    }
                }
                completedAbruptly = false;
            } finally {
                processWorkerExit(w, completedAbruptly);  // Worker结束后的处理逻辑
            }
        }
        ```

        * `getTask()`

          核心线程会直到成功获取任务才退出，而非核心线程会直到超时才退出

          ```java
          private Runnable getTask() {
              boolean timedOut = false;  // 最后一次poll()时是否超时
              for (;;) {
                  int c = ctl.get();
                  int rs = runStateOf(c);
          
                  // 1.SHUTDOWN状态且等待队列为空 
                  // 2.STOP及以上状态
                  // 以上两种情况都表示不再获取新任务
                  // 此时线程的状态为SHUTDOWN或STOP，即调用了shutdown()或shutdownNow()方法
                  if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                      decrementWorkerCount();
                      return null;  // #1
                  }
          
                  // 1.RUNNING状态
                  // 2.SHUTDOWN状态但队列中还有任务 #3
                  // 以上两种情况会继续往下执行
                  int wc = workerCountOf(c);
          
                  // 表示是否会销毁空闲的核心线程 或 超出核心线程数的线程
                  boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
          
                  // 1.wc数超过最大线程数 或 有空闲线程且已超过空闲时间
                  // 2.wc>1 或 等待队列为空
                  // 满足以上两种情况时wc-1
                  if ((wc > maximumPoolSize || (timed && timedOut))
                      && (wc > 1 || workQueue.isEmpty())) {
                      if (compareAndDecrementWorkerCount(c))
                          return null;  // #2
                      continue;  // 循环+CAS的结构
                  }
           
                  try {
                      Runnable r = timed ?
                          // 若队列为空则返回null，超时时间即为预先设定的非核心线程的空闲时间
                          workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                      	// 若队列为空，则在此阻塞
                      	workQueue.take();
                      if (r != null)
                          return r;
                      timedOut = true;  // 再次执行for循环，并在#2处结束
                  } catch (InterruptedException retry) {
                      // 执行中断的线程会抛出异常，在此捕获！
                      timedOut = false;  // 再次执行for循环，并在#1处结束
                  }
              }
          }
          ```
          
        * `processWorkerExit()`

            1. `completedTaskCount += w.completedTasks`：统计线程执行完成的数量

            2. `workers.remove(w)`：销毁此Worker！

            3. `tryTerminate()`：尝试终止线程池

            4. `addWorker(null, false)`：调用`shutdown()`时，如果队列仍不为空则会加一个空Worker

               > 当核心线程正常执行到此步时，队列中肯定没有任务了，所以通常都是添加失败

    * `shutdown()`

      线程池变为SHUTDOWN状态，不再接收新任务，但会处理完等待队列中和正在运行的任务

      有一种情况是所有线程都在执行任务，队列中也还有等待的任务，那么执行此方法只是改变了线程池的状态，而没有中断任何一个线程

      ```java
      public void shutdown() {
          final ReentrantLock mainLock = this.mainLock;
          mainLock.lock();
          try {
              checkShutdownAccess();
              advanceRunState(SHUTDOWN);	// 设置线程池状态为SHUTDOWN
              interruptIdleWorkers();		// 调用interruptIdleWorkers(false)方法
              onShutdown();
          } finally {
              mainLock.unlock();
          }
          tryTerminate();  // 尝试终止线程池
      }
      ```
      
      * `interruptIdleWorkers()`
        
           Worker锁：只有正在执行`getTask()`的Worker没有加锁（即空闲线程）
           
          ```java
          private void interruptIdleWorkers(boolean onlyOne) {
              final ReentrantLock mainLock = this.mainLock;
              mainLock.lock();
              try {
                  for (Worker w : workers) {
                      Thread t = w.thread;
                      // 因为正在执行任务的线程都会获取相应的Worker锁，tryLock()返回false
                      // 所以正在执行任务的线程是无法标记中断的
                      if (!t.isInterrupted() && w.tryLock()) {
                          try {
                              t.interrupt();  // 标记中断，在getTask()的catch块中被捕获
                          } catch (SecurityException ignore) {
                          } finally {
                              w.unlock();
                          }
                      }
                      if (onlyOne)
                          break;
                  }
              } finally {
                  mainLock.unlock();
              }
          }
          ```
      
      * `shutdownNow()`

        线程池变为STOP状态，不再接收新任务，也不会处理等待队列中的任务，还会尝试中止正在执行的任务

        ```java
        public List<Runnable> shutdownNow() {
            List<Runnable> tasks;
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                checkShutdownAccess();
                advanceRunState(STOP);	// 设置线程池状态为STOP
                interruptWorkers();		// 遍历等待队列，调用interruptIfStarted()方法
                tasks = drainQueue();	// 返回等待队列中未执行的任务
            } finally {
                mainLock.unlock();
            }
            tryTerminate();  // 尝试终止线程池
            return tasks;
        }
        ```

        * `interruptIfStarted()`
          
            ```java
            void interruptIfStarted() {
                Thread t;
                // AQS的state>0表示线程已经开始执行任务了
                if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                    try {
                        t.interrupt();  // 标记中断
                    } catch (SecurityException ignore) {
                    }
                }
            }
            ```

      * `tryTerminate()`

        ```java
        final void tryTerminate() {
            for (;;) {
                int c = ctl.get();
                // 1.RUNNING状态
                // 2.TIDYING或TERMINATED状态（有其它线程正在终止线程池）
                // 3.SHUTDOWN状态且等待队列不为空
                // 以上三种情况直接略过
                if (isRunning(c) ||
                    runStateAtLeast(c, TIDYING) ||
                    (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                    return;
                // 此时线程池的状态为SHUTDOWN或STOP，即调用了shutdown()或shutdownNow()方法
                // 因为正在运行任务的线程是无法标记中断的，当这些线程再次执行getTask()时，最后会阻塞
                // 所以需要对这些在批量标记中断后才阻塞的线程再次进行中断
                if (workerCountOf(c) != 0) {
                    // 中断传播：
                    // ABCDE五条线程，AB空闲被成功标记中断，CDE执行后发现还有一个任务，通过#3处判断
                    // C获得任务，但DE只能进行阻塞。C执行完最后这个任务后，在#1处结束，执行销毁逻辑
                    // 最终C执行到此步，对D成功标记中断，D也开始执行销毁逻辑，E同上
                    interruptIdleWorkers(ONLY_ONE);
                    return;
                }
        
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 设置为TIDYING状态
                    if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                        try {
                            terminated();
                        } finally {
                            // 设置为TERMINATED状态
                            ctl.set(ctlOf(TERMINATED, 0));
                            // 对应awaitTermination() ==> termination.awaitNanos(nanos)
                            termination.signalAll();
                        }
                        return;
                    }
                } finally {
                    mainLock.unlock();
                }
            }
        }
        ```

      * `awaitTermination()`

        一段时间后查询线程池状态

        在执行`shutdown()` / `shutdownNow()`方法后调用此方法来获知线程池是否已终止

        ```java
        public boolean awaitTermination(long timeout, TimeUnit unit)
            // ...
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);  // 阻塞，直到被唤醒或者超时
            }
            // ...
        }
        ```

      > 线程池的关键点：
      >
      > 七个参数、四种拒绝、四步流程、Worker与线程、循环获取队列任务、线程销毁、终止线程池

      > Q：线程运行时执行中断，是否会抛出异常？
      >
      > A：与线程的具体状态有关（Thread类中有一个内部枚举类State）
      >
      > ​	NEW / TERMINATED		无影响
      >
      > ​	RUNNABLE / BLOCKED	设置中断标志位但不会强制终止线程，对于线程的终止权利依靠代码判断
      >
      > ​	WAITING（TIMED_WAITING）	非常敏感，会抛出异常并清空中断标志位

      > 设置合理的核心线程数：
      >
      > 线程等待时间所占比例越高，需要越多的线程。线程CPU时间所占比例越高，需要越少的线程
      >
      > 1. IO密集型：2N*cpu* + 1 或 使用`newCachedThreadPool()`方法创建
      > 2. 计算密集型：N*cpu* + 1
      >    由于线程有执行栈等内存消耗，创建过多的线程不会加快计算速度，反而会消耗更多的内存空间；
      >    另一方面线程过多，频繁切换线程上下文也会影响线程池的性能
      > 3. 混合型：N*cpu* ∗ U*cpu* ∗ ( 1 + W / C )
      >    U*cpu*表示CPU利用率的期望值，W / C是等待时间与计算时间的比值

    【参考】

    <http://www.cnblogs.com/trust-freedom/p/6594270.html>

    <http://www.cnblogs.com/trust-freedom/p/6681948.html>

    <http://www.cnblogs.com/trust-freedom/p/6693601.html>

* Executors类：创建ThreadPoolExecutor的工厂类

  ```java
  public class Executors {
      // 单线程：只有一个核心线程
      public static ExecutorService newSingleThreadExecutor() {...}
      // 固定数目：核心线程数与最大线程数相同
      public static ExecutorService newFixedThreadPool(int nThreads) {...}
      // 缓存型：只创建空闲时间为60s的非核心线程
      // 用于耗时较短的任务，适合任务处理速度 > 任务提交速度，才能保证不会不断创建新的进程
      public static ExecutorService newCachedThreadPool() {...}
      // 定时型
      public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)｛...}
  }
  ```
  
  >内存溢出：前两种可能因为线程都堆积在LinkedBlockingQueue队列而导致OOM，后两种可能因为创建了Integer.MAX_VALUE数量的线程而导致OOM

##### ForkJoin

核心思想：分治算法，把一个大任务切分成N个小任务<u>并行执行</u>，最终再聚合结果

组成：ForkJoinPool、WorkQueue（工作队列）、ForkJoinWorkerThread（工作线程）、ForkJoinTask

特点：当某个有工作线程的队列的任务执行完后，还会从任意一个队列的队尾窃取任务执行

​			奇数下标的队列有对应的工作线程，<u>保存</u>并<u>执行</u>内部提交的任务；偶数下标的队列只<u>保存</u>外部提交的任务

> v1.8的ParallelStream的底层使用的就是ForkJoin

<img src="http://img.miilnvo.xyz/aqyya.png" alt="aqyya" style="zoom: 25%;" />

* ForkJoinPool类

  ```java
  public class ForkJoinPool extends AbstractExecutorService {
      volatile WorkQueue[] workQueues;  // 奇数下标存内部提交的任务，偶数下标存外部提交的任务
      
      // 双向链表结构
      @sun.misc.Contended
      static final class WorkQueue {
          volatile int base;  // 窃取任务的下标，因为会有其它线程同时获取，所以需要加volatile
          int top;            // 添加和执行任务的下标，只由对应的线程执行，所以不需要加volatile
          ForkJoinTask<?>[] array;           // 保存任务的数组
          final ForkJoinWorkerThread owner;  // 只有奇数下标才有对应的工作线程
      }
      
      // 【外部提交任务的方法】
      // 1.同步+有结果:异步执行task的compute()方法，一直阻塞直到执行结束，最后返回结果
      public <T> T invoke(ForkJoinTask<T> task) {...}
      // 2.异步+无结果:异步执行task的compute()方法，不会阻塞，无返回结果
      public void execute(ForkJoinTask<?> task) {...}
      // 3.异步+有结果:异步执行task的compute()方法，不会阻塞，通过ForkJoinTask(Future)获取结果
      public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {...}
      
      final void runWorker(WorkQueue w) {
          w.growArray();  // 初始化WorkQueue的array数组
          int seed = w.hint;
          int r = (seed == 0) ? 1 : seed;
          for (ForkJoinTask<?> t;;) {
              if ((t = scan(w, r)) != null)  // 偷取任务
                  w.runTask(t);              // 最终会执行task的compute()方法
              else if (!awaitWork(w, r))     // 如果队列中没有任务，则进行阻塞
                  break;                     // 结束后工作线程会被注销
              r ^= r << 13; r ^= r >>> 17; r ^= r << 5;
          }
      }
      
      private boolean awaitWork(WorkQueue w, int r) {
          // ...
          U.park(false, parkTime);
          // ...
      }
  }
  ```

* ForkJoinWorkerThread类

  ```java
  public class ForkJoinWorkerThread extends Thread {
      final ForkJoinPool pool;                 // 当前工作线程所在的线程池
      final ForkJoinPool.WorkQueue workQueue;  // 当前工作线程所对应的工作队列
      
      public void run() {
          // ...
          pool.runWorker(workQueue);
          // ...
      }
  }
  ```
  
* ForkJoinTask抽象类

  ```java
  public abstract class ForkJoinTask<V> implements Future<V>, Serializable {
      volatile int status;
      
      // 【内部提交任务的方法】
      // 1.创建新线程异步执行当前任务的compute()方法（为了不浪费当前线程所以不推荐此方法）
      public fianl ForkJoinTask<V> fork() {...}
      // 2.执行当前任务的compute()方法
      public final V invoke() {...}  
      // 3.t1任务执行compute()方法，t2任务执行fork()方法
      public static void invokeAll(ForkJoinTask<?> t1, ForkJoinTask<?> t2) {...}
  }
  
  public abstract class RecursiveTask<V> extends ForkJoinTask<V> {
      V result;
      protected abstract V compute();
  }
  
  public abstract class RecursiveAction extends ForkJoinTask<Void> {
      protected abstract void compute();
  }
  ```

  使用方法：实现RecursiveTask / RecursiveAction抽象类，重写`compute()`方法

  ```java
  protected Long compute() {
      if ( /*任务足够小*/ ) {
          // 直接计算并返回结果
      } else {
          // 1.拆分成N个子任务
          // 2.调用子任务的fork()方法进行计算（迭代）
          // 3.调用子任务的join()方法返回结果
      }
  }
  ```
  
  
  计算400个随机数的和：创建4条线程，每条线程计算100个数，最后再加起来
  
  <img src="http://img.miilnvo.xyz/czoo7.png" alt="czoo7" style="zoom:85%;" />
  
  ```java
  public void demo() {
      long[] array = initArray(400);  // 初始化一个大小为400的数组
      ForkJoinPool forkJoinPool = new ForkJoinPool(4);  // 设置四条工作线程
      SumTask sumTask = new SumTask(array, 0, array.length);
  
      long startTime = System.currentTimeMillis();
      Long result = forkJoinPool.invoke(sumTask);  // 外部提交任务的第一种方法，阻塞直到执行结束
      long endTime = System.currentTimeMillis();
  
      System.out.println("Fork/join sum: "+result+" in "+(endTime - startTime)+" ms.");
  }
  
  class SumTask extends RecursiveTask<Long> {
      static final int THRESHOLD = 100;  // 拆分任务的临界值
      long[] array; int start; int end;
  
      SumTask(long[] array, int start, int end) {
          this.array = array; this.start = start; this.end = end;
      }
  
      @Override
      protected Long compute() {
          // 如果任务拆分到了最小，则直接计算
          if (end - start <= THRESHOLD) {
              long sum = getSum(start, end, array);
              System.out.println(String.format("%s 计算：%d~%d = %d", ...);
              return sum;
          }
          // 否则就拆分任务
          int middle = (end + start) / 2;
          System.out.println(String.format("%s 拆分task：%d~%d ==> %d~%d, %d~%d", ...);
  
          SumTask st1 = new SumTask(this.array, start, middle);
          SumTask st2 = new SumTask(this.array, middle, end);
          invokeAll(st1, st2);
          Long result1 = st1.join();  // 阻塞获取返回值
          Long result2 = st2.join();
          Long result = result1 + result2;
          System.out.println("%s 结果：%d + %d ==> %d", ...);
          return result;
      }
  }
  ```
  
  <img src="http://img.miilnvo.xyz/azoho.jpg" alt="azoho" style="zoom:50%;" />

【参考】

<https://www.jianshu.com/p/2a6558ae833e?from=timeline&isappinstalled=0>

<https://www.jianshu.com/p/f777abb7b251>

<https://blog.csdn.net/u010841296/article/details/83963637>

<http://xieli.leanote.com/post/Java%E7%9A%84Fork-Join%E6%A1%86%E6%9E%B6>

##### Thread

* ThreadLocal

  ![z1ort](http://img.miilnvo.xyz/z1ort.png)

  Thread拥有一个ThreadLocalMap属性，ThreadLocalMap拥有一个Entry数组，Entry以ThreadLocal当Key

  使用场景：只在当前线程内使用的变量，且变量的生命周期与线程相同

  > 由于ThreadLocalMap的生命周期和Thread相同长，如果不及时删除对象，就有可能造成OOM，最好每次使用完后都调用`remove()`方法

  ```java
  public void demo() {
      ThreadLocal<Object> l1 = new ThreadLocal<>();
      l1.set("A");
      l1.set("B");
      System.out.println(l1.get());  // 结果是B
      
      ThreadLocal<Object> l2 = new ThreadLocal<>();
      l2.set("C");
      System.out.println(l2.get());  // 结果是C
  }
  ```

  ```java
  public class Thread implements Runnable {
  	ThreadLocal.ThreadLocalMap threadLocals = null;
  }
  ```

  ```java
  public class ThreadLocal<T> {
      
      static class ThreadLocalMap {
          private Entry[] table;
          
          // ThreadLocalMap使用ThreadLocal的弱引用作为Key
          // 优点是当ThreadLocal的强引用失效时Key就会被回收，对应的Value会在下一次调用相关方法时清除
          static class Entry extends WeakReference<ThreadLocal<?>> {
              Object value;
              Entry(ThreadLocal<?> k, Object v) {
                  super(k);
                  value = v;
              }
          }
      }
      
      public void set(T value) {
          Thread t = Thread.currentThread();
          ThreadLocalMap map = getMap(t);
          if (map != null)
              map.set(this, value);
          else
              createMap(t, value);
      }
      
      public T get() {
          Thread t = Thread.currentThread();
          ThreadLocalMap map = getMap(t);  // 获取当前线程拥有的ThreadLocalMap
          if (map != null) {
              // this对象(ThreadLocalMap)通过哈希函数获得Entry数组的下标
              ThreadLocalMap.Entry e = map.getEntry(this);
              if (e != null) {
                  @SuppressWarnings("unchecked")
                  T result = (T)e.value;
                  return result;
              }
          }
          // t.threadLocals = new ThreadLocalMap(this, firstValue);
          return setInitialValue();
      }
  }
  ```

  【参考】

  <https://blog.csdn.net/bntx2jsqfehy7/article/details/78315161>

* ThreadLocalRandom（v1.7）

  java.util.Random类使用AtomicLong作为SEED，每获得一次随机数都要用CAS操作更新这个种子

  ```java
  public class Random implements java.io.Serializable {
      // 多个线程要共享一个Random对象，才可以实现多个线程有唯一的随机数
      private final AtomicLong seed;
      
      protected int next(int bits) {
          long oldseed, nextseed;
          AtomicLong seed = this.seed;
          do {
              oldseed = seed.get();
              nextseed = (oldseed * multiplier + addend) & mask;
          } while (!seed.compareAndSet(oldseed, nextseed));  // 高并发环境下，大量线程会自旋重试
          return (int)(nextseed >>> (48 - bits));
      }
  }
  ```

  java.util.concurrent.ThreadLocalRandom类只保存初始的SEED，由每条线程各自保存自己的SEED

  初始化时，ThreadLocalRandom的初始种子为SEED-0，F1是生成新SEED的函数，F2是生成随机数的函数

  线程T1使用时：T1.SEED-0 == F1 ==> T1.SEED-1 == F2 ==> T1.Random-1
  
  线程T2使用时：T2.SEED-0 == F1 ==> T2.SEED-1 == F2 ==> T2.Random-1
  
  线程T1再使用时：T1.SEED-2 == F2 ==> T1.Random-2
  
  > T1.SEED-0 + SEEDER_INCREMENT = T2.SEED-0
  >
  > T1.SEED-1 + GAMMA = T1.SEED-2
  
  ```java
  public void demo() {
      for (int i = 0; i < 10; i++) {
          new Thread(() -> {
              long random = ThreadLocalRandom.current().nextInt();
          }).start();
      }
  }
  ```
  
  ```java
  public class Thread implements Runnable {
      @sun.misc.Contended("tlr")
      long threadLocalRandomSeed;  // SEED
  
      @sun.misc.Contended("tlr")
      int threadLocalRandomProbe;  // 0表示未初始化，1表示已初始化
  
      @sun.misc.Contended("tlr")
      int threadLocalRandomSecondarySeed;  // 二级SEED
  }
  ```
  
  ```java
  public class ThreadLocalRandom extends Random {
      static final ThreadLocalRandom instance = new ThreadLocalRandom();  // 单例模式
      private static final AtomicLong seeder = new AtomicLong(initialSeed());
      
      private static final long SEEDER_INCREMENT = 0xbb67ae8584caa73bL;
      private static final long GAMMA = 0x9e3779b97f4a7c15L;
  
      // 默认使用当前时间来初始化seeder，即SEED-0
      private static long initialSeed() {
          // ...
          return (mix64(System.currentTimeMillis())
                  mix64(System.nanoTime()));
      }
  
      public static ThreadLocalRandom current() {
          if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
              // long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT)); F1函数
              localInit(); 
          return instance;
      }
  
      final long nextSeed() {
          Thread t; long r;
          UNSAFE.putLong(t = Thread.currentThread(), SEED,
                         r = UNSAFE.getLong(t, SEED) + GAMMA);  // F2函数
          return r;
      }
  }
  ```
  
  【参考】
  
  <https://pzh9527.iteye.com/blog/2428066>




#### 原子类

* AtomicXXX

  读：volatile修饰value属性

  写：CAS操作

* LongAdder（v1.8）

  空间换时间的思想，通过不同的线程来操作各自的Cell，分散单一热点

  用于高并发的计数器

  > AtomicLong等原子类在高并发场景下可能会有很多线程同时自旋重试，造成不必要的开销

  > AtomicLong VS LongAdder
  >
  > 单线程：前者性能略高于后者
  >
  > 多线程：后者性能是前者的数倍甚至十倍

  ![cjg55](http://img.miilnvo.xyz/cjg55.jpg)

  ```java
  abstract class Striped64 extends Number {
      transient volatile long base;      // 没有竞争的情况下，通过CAS累加到base
      transient volatile Cell[] cells;   // 有竞争的情况下，通过Hash计算下标后再保存值
      transient volatile int cellsBusy;  // 用于Cell数组创建和扩容的锁
      
      @sun.misc.Contended  // 消除伪共享
      static final class Cell {
          volatile long value;
      }
      
      // 惰性求值，如果此时有线程再添加新值，可能会造成结果不准确
      // 更适用于不需要每次增加都返回当前值的场景
      public long sum() {
          // return sum = base + ∑[0~n]cells
      }
  } 
  ```

  ```java
  public class LongAdder extends Striped64 implements Serializable {
      public void add(long x) {
          Cell[] as; long b, v; int m; Cell a;
          // 首先通过CAS对base进行累加，如果失败则准备使用Cell数组
          // 在非高并发场景下，不需要初始化Cell数组，目的是为了节省空间，使其性能与AtomicLong相同
          if ((as = cells) != null || !casBase(b = base, b + x)) {
              boolean uncontended = true;
              if (as == null || (m = as.length - 1) < 0 ||
                  (a = as[getProbe() & m]) == null ||
                  !(uncontended = a.cas(v = a.value, v + x)))
                  longAccumulate(x, null, uncontended);
          }
      }
      
      final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
          // 0.先初始化当前线程的probe属性，并把它当作Hash值
          for(;;) {
              // 1.cells数组已经正常初始化
                  // 1-1.通过Hash值计算下标，当前下标为null => new cell
                  // 1-2.操作此下标时存在竞争 => 重新计算Hash值
                  // 1-3.通过CAS累加cell的value => 成功则退出for循环
                  // 1-4.数组长度达到最大值或其它线程进行了扩容 => 重新计算Hash值
                  // 1-5.对cells数组进行扩容 => 再次进入循环
              // 2.cells数组未初始化 => 进行初始化
              // 3.cells数组未初始化且cellbusy=1 => 通过CAS对base进行累加
          }
      }
  }
  ```

  > 另一种观点：使用自旋锁时有个隐含的假设就是冲突的概率相当低，如果实际的运行环境中一个原子操作都会有那么多的冲突，那么使用单线程的解决方案可能更有效率。如果实际的运行环境中原子操作没有太多的冲突，那么使用基于CAS的原子操作基本上也能满足要求

  
  【参考】
  
  https://www.jianshu.com/p/dddb531e403c
  
  https://blog.csdn.net/zqz_zqz/article/details/70665941



---

#### 其它

##### 缓存一致性协议（MSEI）

使用高速缓存（L1/L2/L3）来解决内存与CPU的处理速度存在差异的问题

缓存系统中是以缓存行为单位存储的，一个64字节的缓存行理论上可以存储8个Long型变量的值

* 缓存行的四种状态

  修改（M，Modified）：仅当前处理器拥有该缓存行，并且缓存行被修改过了，与主内存不一致

  独占（E，Exclusive）：仅当前处理器拥有该缓存行，并且没有修改过，与主内存一致

  共享（S，Share）：多个处理器都拥有该缓存行，每个处理器都没有修改过，与主内存一致

  无效（I，Invalid）：缓存行被其他处理器修改过，该值不是最新值，需要读取主内存的最新值

  > X进行远程读操作时，如果发现其它缓存行已经有需要的数据了，那么其他缓存行需要先刷新到主内存（M=>E=>S），X读取后也变为S状态
  >
  > X进行远程写操作时，如果缓存行都为S状态，那么X变为E状态，其它缓存行变为I状态（RFO指令），更改后X变为M状态

* 伪共享

  如果一个缓存行被多个线程共享，哪怕多个线程操作的变量不同，写操作时也会使其它缓存行变为I状态，结果是使用了大量的RFO指令，影响性能

  优化方案：

  1. v1.6版本之前：填充无用字段p1、p2、p3……，例如一个long类型的属性要加六个long类型的无用字段

  2. v1.8版本之后：在类上加`@sun.misc.Contended`注解，启动时加上JVM参数`-XX:-RestrictContended`

  > ForkJoinPool类中有WorkQueue[]属性，因为会有多个线程同时操作此双向链表队列，所以WorkQueue类就加上了此注解

  > 测试结果：4核4线程，每条线程都操作同一个数组里的不同对象，总耗时减少了40%

> volatile是一个上层表达意图的"抽象"，而MESI是为了实现这个抽象，在某种特定情况下使用的其中一种具体实现方案。CPU的缓存有很多种，MESI以及RFO指令只是保证了L1/L2/L3的一致性，所以依然会有volatile中提到的"工作内存在更新变量副本后并不会刷新其它工作内存的值"问题

【参考】

<https://blog.csdn.net/yinwenjie/article/details/83069483>

<https://www.cnblogs.com/cyfonly/p/5800758.html>

<https://www.cnblogs.com/yanlong300/p/8986041.html>





<https://segmentfault.com/blog/ressmix_multithread?page=2>