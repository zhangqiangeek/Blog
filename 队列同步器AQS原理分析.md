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
AQS通过内置的FIFO队列来完成资源获取线程的排队工作，队列的节点是基于内部静态类Node实现的。
> AQS对于等待队列的支持是完全的，完全屏蔽了被阻塞的线程的入队和出队操作的实现细节；而对state变量的维护只是提供了部分支持，需要开发者复写特定的方法才能正常工作。

关于对等待队列的理解，索性直接把源码中的注释贴出来，大家自行理解。

```
/**
     * Wait queue node class.
     *
     * <p>The wait queue is a variant of a "CLH" (Craig, Landin, and
     * Hagersten) lock queue. CLH locks are normally used for
     * spinlocks.  We instead use them for blocking synchronizers, but
     * use the same basic tactic of holding some of the control
     * information about a thread in the predecessor of its node.  A
     * "status" field in each node keeps track of whether a thread
     * should block.  A node is signalled when its predecessor
     * releases.  Each node of the queue otherwise serves as a
     * specific-notification-style monitor holding a single waiting
     * thread. The status field does NOT control whether threads are
     * granted locks etc though.  A thread may try to acquire if it is
     * first in the queue. But being first does not guarantee success;
     * it only gives the right to contend.  So the currently released
     * contender thread may need to rewait.
     *
     * <p>To enqueue into a CLH lock, you atomically splice it in as new
     * tail. To dequeue, you just set the head field.
     * <pre>
     *      +------+  prev +-----+       +-----+
     * head |      | <---- |     | <---- |     |  tail
     *      +------+       +-----+       +-----+
     * </pre>
     *
     * <p>Insertion into a CLH queue requires only a single atomic
     * operation on "tail", so there is a simple atomic point of
     * demarcation from unqueued to queued. Similarly, dequeuing
     * involves only updating the "head". However, it takes a bit
     * more work for nodes to determine who their successors are,
     * in part to deal with possible cancellation due to timeouts
     * and interrupts.
     *
     * <p>The "prev" links (not used in original CLH locks), are mainly
     * needed to handle cancellation. If a node is cancelled, its
     * successor is (normally) relinked to a non-cancelled
     * predecessor. For explanation of similar mechanics in the case
     * of spin locks, see the papers by Scott and Scherer at
     * http://www.cs.rochester.edu/u/scott/synchronization/
     *
     * <p>We also use "next" links to implement blocking mechanics.
     * The thread id for each node is kept in its own node, so a
     * predecessor signals the next node to wake up by traversing
     * next link to determine which thread it is.  Determination of
     * successor must avoid races with newly queued nodes to set
     * the "next" fields of their predecessors.  This is solved
     * when necessary by checking backwards from the atomically
     * updated "tail" when a node's successor appears to be null.
     * (Or, said differently, the next-links are an optimization
     * so that we don't usually need a backward scan.)
     *
     * <p>Cancellation introduces some conservatism to the basic
     * algorithms.  Since we must poll for cancellation of other
     * nodes, we can miss noticing whether a cancelled node is
     * ahead or behind us. This is dealt with by always unparking
     * successors upon cancellation, allowing them to stabilize on
     * a new predecessor, unless we can identify an uncancelled
     * predecessor who will carry this responsibility.
     *
     * <p>CLH queues need a dummy header node to get started. But
     * we don't create them on construction, because it would be wasted
     * effort if there is never contention. Instead, the node
     * is constructed and head and tail pointers are set upon first
     * contention.
     *
     * <p>Threads waiting on Conditions use the same nodes, but
     * use an additional link. Conditions only need to link nodes
     * in simple (non-concurrent) linked queues because they are
     * only accessed when exclusively held.  Upon await, a node is
     * inserted into a condition queue.  Upon signal, the node is
     * transferred to the main queue.  A special value of status
     * field is used to mark which queue a node is on.
     *
     * <p>Thanks go to Dave Dice, Mark Moir, Victor Luchangco, Bill
     * Scherer and Michael Scott, along with members of JSR-166
     * expert group, for helpful ideas, discussions, and critiques
     * on the design of this class.
     */
```

#### 模板方法

AQS中提供的模板方法如下：

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

### 实现一个独占锁
todo
### 源码分析
todo

本文参考:田守枝的博客和《并发编程的艺术》