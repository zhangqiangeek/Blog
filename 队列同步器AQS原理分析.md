---
   title: 队列同步器AQS原理分析
   date: 2019-05-08 14:11:00
   tags: java
---

队列同步器AbstractQueuedSynchronizer（以下简称AQS）是用来构建锁或者其他同步组件的基础框架，使用一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作，并发包的作者Doug Lea期望它能够成为实现大部分同步需求的基础。

在java.util.concurrent.locks包中的很多类都是基于AQS实现的，例如：CountDownLatch，CyclicBarrier，Semaphore，重入锁ReentrantLock和读写锁(ReentrantReadWriteLock)。可以说没有弄懂AQS，就没有真正的掌握java并发包。AQS在LockSupport和Unsafe类的基础上实现的，主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。
<!-- more -->
### 功能介绍
AQS的设计是基于模板方法模式，使用者需要继承同步器并重写指定的方法，随后将同步器在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

AQS作为构建同步组件的基础框架，支持独占式地获取同步状态，也支持共享式的获取同步状态。不论是独占还是共享，使用的过程本质上都是获取许可与释放许可，而获取许可和释放许可中最关键是对于许可数量的维护和等待队列的维护。
#### 许可的维护

AQS使用一个int类型的state变量表示同步状态，初始值默认为0，线程每获取一个许可，state变量+1，表示使用了几个许可，对于state变量的维护，必须由开发人员自己来实现。

```
    /**
     * The synchronization state.
     */
    private volatile int state;
```

重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。
- getState():获取当前同步状态
- setState(int new State):设置当前同步状态
- compareAndSerState(int expect,int update):使用CAS设置当前状态，CAS能保证状态设置的原子性。

三个方法的代码如下：

```
protected final int getState() {
        return state;
    }
    
protected final void setState(int newState) {
        state = newState;
    }

protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
#### 等待队列的维护
AQS通过内置的FIFO队列来完成资源获取线程的排队工作，队列的节点是基于内部静态类Node实现的。当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构构造成一个节点并将其加入同步队列，同时阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次获取同步状态。


> AQS对于等待队列的支持是完全的，完全屏蔽了被阻塞的线程的入队和出队操作的实现细节；而对state变量的维护只是提供了部分支持，需要开发者复写特定的方法才能正常工作。


同步队列中的节点保存了获取同步状态失败的线程引用、等待状态、前驱和后继节点，节点的属性类型、名称和描述等。Node源码如下：

```
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
        Node nextWaiter;
        
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

- int waitStatus: 等待状态
- Node prev: 前驱节点，当节点被加入到同步队列时被设置。、
- Node  next: 后继节点
- Node nextWaiter: 等待队列中的后继节点。如果当前节点是共享的，那么这个字段是一个SHARED常量，也就是说节点类型和等待队列中的后继节点公用同一个字段
- Thread thread:获取同步状态的线程

等待队列的示意图如下图所示：
![同步队列](http://pic.evilhex.com/2019-05-10-同步队列.jpg)

#### AQS提供的模板方法

AQS中提供的模板方法如下，程序员可以根据自身需求自己实现

方法名称 | 描述
--------- | -------------
void acquire(int arg) | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用重写的tryAcquire(int arg)方法
void acquireInterruptibly(int arg) |与tryAcquire(int arg)相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException
boolean tryAcquireNanos(int arg, long nanosTimeout)|在acquireInterruptibly(int arg)基础上增加了超时限制，如果当前线程在超时实现内没有获取到同步状态，那么将会返回false，如果获取到了就返回true
boolean release(int arg)|独占式释放同步状态，该方法会在释放同步状态后，将同步队列中第一个节点包含的线程唤醒
void acquireShared(int arg)|共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态。
void acquireSharedInterruptibly(int arg)|与void acquireInterruptibly(int arg)相同，该方法响应中断
boolean tryAcquireSharedNanos(int arg, long nanosTimeout)|在acquireSharedInterruptibly(int arg)的基础上增加了超时限制。
boolean releaseShared(int arg)| 共享式释放同步状态
Collection<Thread> getQueuedThreads()|获取等待在同步队列上的线程集合。

### 原理分析
#### 独占式同步状态获取与释放
通过调用同步器的acquire(int arg)方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续线程进行中断操作时，线程不会从同步队列中移除，代码如下所示。

```
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
tryAcquire(arg)方法可以保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点并通过addWaiter(Node.EXCLUSIVE), arg)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node,int arg)方法以死循环的方式获取同步状态。
三个关键方法的代码以及注释如下。
```
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

```
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // 只有前驱节点是头结点才能尝试获取同步状态
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

当前线程获取了同步状态并执行完成相应逻辑之后，就需要释放同步状态，以便使后续节点能够继续获取同步状态。同步器中的release(int arg)方法可以释放同步状态，改方法在释放了同步状态之后，会唤醒后继节点，代码如下。

```
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

```
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
#### 共享式同步状态获取与释放
既然是共享式获取，那么与独占式获取的主要区别就在于同一时刻是否有多个线程同时获取到同步状态。JDK中的读写锁就是一个共享式同步状态获取的典型例子。这个部分的核心代码如下所示。

```
public final void acquireShared(int arg) {
        // tryAcquireShared方法返回值大于0时，表示能够获取到同步状态。
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

```
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
共享式释放同步状态是通过调用releaseShared来实现的，在释放同步状态之后，将会唤醒后续处于等待状态的节点。

```
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

本文参考:田守枝的博客和《并发编程的艺术》