# AQS

此文是`AbstractQueuedSynchronizer`的概述，详细解析参考[ReentrantLock](reentrant-lock.md)和[CountDownLatch](count-down-latch.md)

- [AQS](#aqs)
  - [AbstractQueuedSynchronizer](#abstractqueuedsynchronizer)
  - [实例分析](#实例分析)
  - [Queue](#queue)
  - [AbstractQueuedSynchronizer.Node](#abstractqueuedsynchronizernode)
  - [Node.waitStatus](#nodewaitstatus)
    - [CANCELLED](#cancelled)
    - [SIGNAL](#signal)
    - [PROPAGATE](#propagate)
    - [CONDITION](#condition)
  - [AbstractQueuedSynchronizer queue and state](#abstractqueuedsynchronizer-queue-and-state)
  - [参考](#参考)

## AbstractQueuedSynchronizer

`java.util.concurrent.locks.AbstractQueuedSynchronizer`

1. `AbstractQueuedSynchronizer` 是一个模板抽象类,封装了算法细节,暴露了很多 `protected` 方法方便子类重写
2. `AbstractQueuedSynchronizer` 是基于 `FIFO` 队列实现的
3. `AbstractQueuedSynchronizer` 中使用 `volatile int state` 来`计数`
4. `AbstractQueuedSynchronizer` 可以实现可以重入锁(或者不可重入锁)的语义，如 `ReentrantLock`
5. `AbstractQueuedSynchronizer` 可以实现共享锁，排他锁的语义，如 `ReentrantReadWriteLock`
6. `AbstractQueuedSynchronizer` 可以实现公平锁，非公平锁的语义

## 实例分析

- [CountDownLatch](count-down-latch.md)
- [ReentrantLock](reentrant-lock.md)

## Queue

`Condition queue And Main queue`

Threads waiting on Conditions use the same nodes, butuse an additional link. Conditions only need to link nodes
in simple (non-concurrent) linked queues because they areonly accessed when exclusively held.  Upon await, a node is
inserted into a `condition queue`.  Upon signal, the node istransferred to the `main queue`.  A special value of status
field is used to mark which queue a node is on.

为什么需要两个 `Queue` ?

`Main Queue`中的线程是获取锁失败在排队`等待`的线程,如果队列前面的线程释放了锁，就需要去获取锁。 `Main Queue` 中的所有线程都在`排队`等待锁的获取。

而 `condition queue` 中的线程是在执行 `Condition.await` 这个方法之后加入的。它的含义是线程不去参加锁的竞争，线程去`休息`了。因此它不能在 `Main Queue` 中（线程不`排队`了）。

而在执行 `Condition.signal` 或者 `Condition.signalAll` 之后才开始去`排队`竞争锁，才需要进入到 `Main Queue` 中。

## AbstractQueuedSynchronizer.Node

```java
// Node 的定义
static final class Node {
 static final Node SHARED = new Node();
 static final Node EXCLUSIVE = null;
 static final int CANCELLED =  1;
 static final int SIGNAL    = -1;
 static final int CONDITION = -2;
 static final int PROPAGATE = -3;
 volatile int waitStatus;
 volatile Node prev;
 volatile Node next;
 volatile Thread thread;
 // condition Queue
 Node nextWaiter;
}
```

## Node.waitStatus

```java
// Node
// waitStatus 是 Node 的成员变量
static final class Node {
    volatile int waitStatus;
}
```

```java
/** waitStatus value to indicate thread has cancelled */
static final int CANCELLED =  1;

/** waitStatus value to indicate successor's thread needs unparking */
static final int SIGNAL    = -1;

/** waitStatus value to indicate thread is waiting on condition */
static final int CONDITION = -2;

/**
 * waitStatus value to indicate the next acquireShared should
 * unconditionally propagate
 */
static final int PROPAGATE = -3;

```

### CANCELLED

线程获取锁失败，比如：线程执行了 `interrupt` 方法,或者获取锁超时，此时正在阻塞的线程就会被唤醒，进入 `CANCELLED` 状态

可以参考 `AbstractQueuedSynchronizer` 的 `cancelAcquire` 方法

### SIGNAL

`AQS`的锁实现是基于`FIFO`队列的，一个线程获取锁资源失败的时候，会把它的前置节点(代码中用prev节点表示)修改成`SIGNAL(-1)`状态
表示其他没有获取锁的线程需要被唤醒（获取锁的线程可通过执行unparkSuccessor方法，唤醒等待的线程。具体逻辑在AbstractQueuedSynchronizer#release方法中）

### PROPAGATE

`PROPAGATE` 状态在 `doReleaseShared` 方法中使用，表示 `waitStatus` 直接从 `0` 修改成 `PROPAGATE`

`doReleaseShared` 的代码逻辑如下：

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

### CONDITION

`CONDITION` 表示进入了等待队列,在执行 `condition.await()` 之后会修改这个状态

代码片段如下：

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

## AbstractQueuedSynchronizer queue and state

AbstractQueuedSynchronizer 最重要的几个变量

`Node head` 和 `Node tail` 用来组成 FIFO queue

`int state` 变量用来对获取锁的线程计数

此外它们都是用 `volatile` 来修饰的

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
```

## 参考

- [AQS](https://juejin.cn/post/6977746796093112333)
- [AQS](https://www.cnblogs.com/waterystone/p/4920797.html)
- [cas and aqs (csdn)](https://blog.csdn.net/u010862794/article/details/72892300)
- [aqs (github)](<https://github.com/CL0610/Java-concurrency/blob/master/08.%E5%88%9D%E8%AF%86Lock%E4%B8%8EAbstractQueuedSynchronizer(AQS)/%E5%88%9D%E8%AF%86Lock%E4%B8%8EAbstractQueuedSynchronizer(AQS).md>)
- [aqs (简书)](https://www.jianshu.com/p/cc308d82cc71)
- [aqs](https://wyj.shiwuliang.com/JAVA%20-%20AQS%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.html)
- [https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
- [https://tech.meituan.com/2018/11/15/java-lock.html](https://tech.meituan.com/2018/11/15/java-lock.html)