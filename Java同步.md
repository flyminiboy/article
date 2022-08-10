什么是线程安全

1. 原子性 同步机制
2. 可见性 volatile 
3. 有序性 指令重排序


### volatile



### synchronized

#### java 

Java 内置锁或监视器锁

```java
public class a {
    
    public void test() {
        synchronized (this) {
            System.out.println("hello");
        }
    }

    public synchronized void b() {
        System.out.println("aaa");
    }
    
}
```


```
  public void test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter      // 关键******
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit        // 关键******
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit         // 关键******
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 6: 0
        line 7: 4
        line 8: 12
        line 9: 22
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class com/example/mockcrash/a, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

```


```
  public synchronized void b();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String aaa
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 12: 0
        line 13: 8

```


#### kotlin 

```kotlin

    private fun testLock() {
        synchronized(this) {
            println("lock")
        }
    }
```

```kotlin
  private final testLock()V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 48 L4
    ALOAD 0
    ASTORE 1
   L5
    ICONST_0
    ISTORE 2
   L6
    ICONST_0
    ISTORE 3
   L7
    ALOAD 1
    MONITORENTER
   L0
    NOP
   L8
   L9
    ICONST_0
    ISTORE 4
   L10
    LINENUMBER 49 L10
    LDC "lock"
    ASTORE 5
   L11
    ICONST_0
    ISTORE 6
   L12
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    ALOAD 5
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/Object;)V
   L13
   L14
    LINENUMBER 50 L14
    NOP
   L15
    GETSTATIC kotlin/Unit.INSTANCE : Lkotlin/Unit;
   L16
    LINENUMBER 48 L16
    ASTORE 3
   L1
    ALOAD 1
    MONITOREXIT
   L17
    GOTO L18
   L2
    ASTORE 3
   L3
    ALOAD 1
    MONITOREXIT
    ALOAD 3
    ATHROW
   L18
   L19
    LINENUMBER 51 L19
    RETURN
   L20
    LOCALVARIABLE $i$a$-synchronized-MainActivity$testLock$1 I L10 L15 4
    LOCALVARIABLE this Lcom/example/camera2/MainActivity; L4 L20 0
    MAXSTACK = 2
    MAXLOCALS = 7
```



用 javap 反编译，可以看到类似片段，利用 monitorenter/monitorexit 对实现了同步的语义

### CAS (compare-and-swap)


偏斜锁
轻量级锁
重量级锁




自旋锁

```java

```







### AQS AbstractQueuedSynchronizer

核心数据结构**等待队列**是 `CLH 队列`的一个变种



#### 等待队列节点类 - Node

```java
static final class Node {
        
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
       
    }
```

```java

    /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;
    
    
    
    /**
     * Attempts to acquire in exclusive mode. This method should query
     * if the state of the object permits it to be acquired in the
     * exclusive mode, and if so to acquire it.
     *
     * <p>This method is always invoked by the thread performing
     * acquire.  If this method reports failure, the acquire method
     * may queue the thread, if it is not already queued, until it is
     * signalled by a release from some other thread. This can be used
     * to implement method {@link Lock#tryLock()}.
     *
     * <p>The default
     * implementation throws {@link UnsupportedOperationException}.
     *
     * @param arg the acquire argument. This value is always the one
     *        passed to an acquire method, or is the value saved on entry
     *        to a condition wait.  The value is otherwise uninterpreted
     *        and can represent anything you like.
     * @return {@code true} if successful. Upon success, this object has
     *         been acquired.
     * @throws IllegalMonitorStateException if acquiring would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * Attempts to set the state to reflect a release in exclusive
     * mode.
     *
     * <p>This method is always invoked by the thread performing release.
     *
     * <p>The default implementation throws
     * {@link UnsupportedOperationException}.
     *
     * @param arg the release argument. This value is always the one
     *        passed to a release method, or the current state value upon
     *        entry to a condition wait.  The value is otherwise
     *        uninterpreted and can represent anything you like.
     * @return {@code true} if this object is now in a fully released
     *         state, so that any waiting threads may attempt to acquire;
     *         and {@code false} otherwise.
     * @throws IllegalMonitorStateException if releasing would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
```





`CLH 锁`通常用于**自旋锁**







#### ReentrantLock

lock

tryLock

unlock

##### 条件变量 `Condition`

await

signal

signalAll

#### CountDownLatch

await

countDown

#### CyclicBarrier

await

#### Semaphore

acquire

tryAcquire

release





