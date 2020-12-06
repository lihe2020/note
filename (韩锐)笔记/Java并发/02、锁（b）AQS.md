### 1. AQS介绍

#### 1.1 AQS说明

AQS是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作，并发包的作者(Doug Lea)期望它能够成为实现大部分同步需求的基础。

#### 1.2 AQS与锁的联系与区别

AQS的子类通过承继的方式来实现相应的抽象方法，并且通过对同步状态的变更来实现锁功能。

**锁是面向使用者的**，它定义了使用者与锁交互的接口(比如可以允许两个线程并行访问)，隐藏了实现细节；**AQS面向的是锁的实现者**， 它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和AQS很好地隔离了使用者和实现者所需关注的领域。

> 这段话拷贝自《Java并发编程的艺术》，总结的多精辟，能说出这话的人肯定是认清了AQS的本质。把它拷贝到这，是因为面试会问到。

### 2. AQS内部工作原理

> 以前，我认为看懂了源码一切都OK，但是面试的过程中发现即使看懂了源码，很多问题也没办法回答上来。想了很久，原因可能有二：
> 
> - 源码太过于细节，虽然了解了每个环节，但是无法串联成流程图
> - 不善于做总结，看了忘，忘了看，拜拜浪费时间
> 
> 基于此，在以后的源码分析前我都会来个工作原理/流程总结。

AQS依赖内部的同步队列来完成同步状态的管理，同步队列是一个双向链表结构。当线程获取锁失败时，AQS会以CAS的方式（compareAndSetTail）将一个包含当前线程的Node节点加入同步队列尾部，在有其他线程释放锁时，队列第一个非虚Node节点对应的线程将会被唤醒并尝试去获取锁，获取成功后会将自己设为首节点，这个过程由于只有一个线程能够获取到锁，所以设置首节点这个步骤并不需要CAS。

处于同步队列中的线程会阻塞自己，如果被（意外）唤醒，它会检测自己是否是第一个非虚节点，是的话获取锁，否的话继续park。

### 3. AQS源码解析

#### 3.1 AQS结构

AQS全称AbstractQueuedSynchronizer，是用来构建锁和其他同步组件的基础框架。它使用了一个int成员变量`state`表示同步状态，通过内置的等待队列来完成资源获取线程的排队工作。

等待队列是一个FIFO的链表结构，存放的是排队等待获取锁的Node集合。每个Node节点自旋式获取前驱Node的状态，只有当前驱Node释放锁后才会尝试获取锁。

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
  
  // 等待队列存放的是排队等待获取锁的Node集合，它是链表结构
  // AQS中的等待队列是CLH的一个变种，CLH是单向的，而这里的等待队列是双向FIFO队列
  // 当第一次往队列中添加元素前会添加一个空的Node节点，称为虚节点，虚节点的next、prev字段均为空
  
  // 等待队列的队首、队尾
  private transient volatile Node head;
  private transient volatile Node tail;
  
  // 0  没有线程持有该锁；
  // 1  有1个线程持有该锁；
  // >1 有1个线程重复持有该锁（重入）；
  private volatile int state;
  
  // 尝试获取独占锁
  protected boolean tryAcquire(int arg) {
  }
  // 独占（排他）模式获取锁
  public final void acquire(int arg) {
  }
  
  // 尝试释放锁，此时其他线程将有机会获取锁
  protected boolean tryRelease(int arg) {
  }
  
  // 释放独占的锁
  public final boolean release(int arg) {
  }
  
  // 获取当前同步状态
  protected final int getState() {
  }
  
  // 设置同步状态
  protected final boolean compareAndSetState(int expect, int update) {
  }
  
  // 是否是当前线程持该锁
  // 只有当前线程持有锁，才能调用条件等待，否则抛IllegalMonitorStateException异常
  protected boolean isHeldExclusively() {
  }
}
```

上述代码中的Node代表着队列中等待的线程，它的结构如下：

```java
static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;

        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;

        // 当前节点在队列中的状态
        // 0， 默认初始值
        // CANCELLED 1，表示线程获取锁的请求已经取消了
        // CONDITION -2, 节点在等待队列中，节点线程等待唤醒
        // PROPAGATE -3，当前线程处在SHARED情况下，该字段才会使用
        // SIGNAL -1，表示下一个Node对应的线程等待获取锁
        volatile int waitStatus;

        // 前驱节点
        volatile Node prev;
        // 后继节点
        volatile Node next;
        // 与节点关联的线程
        volatile Thread thread;

  			 // 指向下一个处于CONDITION状态的节点
        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        // 获取前驱节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        ...
    }
```

#### 3.2 获取锁流程

```java
public final void acquire(int arg) {
    // 1. tryAcquire 尝试获取锁
    // 2. addWaiter 将当前线程以排他的方式加入等待队列
    // 3. acquireQueued 进入队列的线程会不断的获取锁（自旋的方式）
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 添加当前线程到等待队列
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {  //如果pred不为null，代表队列已经初始化过
        node.prev = pred;
        // 设置tail为node，然后将pred的next指向node
        // 注意这里是2步操作，所以可能存在中间节点的next=null的情况
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
  
    // 此时队列可能还没有初始化，进行入队操作
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            // 尝试将head、tail设置为虚节点
            if (compareAndSetHead(new Node()))
                tail = head;
          
            // 注意这里会进入下一次循环，会走到else分支去设置node
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// 不断的尝试：
// 1. 获取锁
// 2. 成功返回、失败park自己
// 3. 醒来后重复执行上述步骤
final boolean acquireQueued(final Node node, int arg) {
    // 是否失败标识
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果node的前驱节点是head并且成功获取锁
            // 则将head替换成node，此时node将会变成一个虚节点，也就是head永远会是一个虚节点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
          
            // 以下可能性会导致代码执行到此处：
            // 1. node不是head的后继
            // 2. node是head后继节点，但其他线程已经到锁
            // 那么尝试park（阻塞）自己
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 如果pred是SIGNAL（等待获取锁）状态，那么继续阻塞吧
        return true;
    if (ws > 0) {  // waitStatus>0是取消状态
        do {
            // 往前遍历，将所有取消的node移除
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        // 对于排它锁来说，如果代码执行到这里，状态一定是0（参见唤醒逻辑）
        // 此时，要么是第一次执行到此（head是虚节点而是默认状态），要么没有成功获取到锁
        // 那么将状态重置为SIGNAL，代表当前节点仍需等待锁释放
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

#### 3.3 释放锁流程

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

// 尝试修改head状态为0，并且唤醒head的后继节点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);  // 将head的状态重置为默认状态
  
    // 如果下一个节点已经取消，那么将其移除，
    // 重复上述步骤，直到遇到第一个没被取消的节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒head的后继节点线程
}
```

#### 3.4 取消锁流程

以`doAcquireInterruptibly`以为例，这里的代码基本上和`acquireQueued`，只是多了线程打断处理逻辑，线程被打断后会进入取消锁的逻辑，详细代码：

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void cancelAcquire(Node node) {
    if (node == null)
        return;
    node.thread = null;
    // 移除所有被取消的前驱节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    Node predNext = pred.next;
    node.waitStatus = Node.CANCELLED;
    // 如果当前节点是尾节点，那么尝试将上一个（非取消状态的）节点设为尾节点
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // 如果当前节点是首节点，那么将下一个节点替换成首节点
        // 如果是非首节点，那么将前一节点与后一节点相连
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // 唤醒后继节点，代码见释放锁流程
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

#### 3.5 ConditionObject源码

ConditionObject是AQS的条件等待队列，`await()`用于释放当前线程持有的锁，然后挂起；`signalAll()`用于尝试唤醒此条件等待队列的所有线程。

ConditionObject内部的也是链表结构，存放处于此等待队列的所有线程：

```java
public class ConditionObject implements Condition, java.io.Serializable {
  private transient Node firstWaiter;
  private transient Node lastWaiter;
  
  // 将链表所有节点追加到AQS的锁等待队列中
  // 加入锁等待队列后才有机会获取锁
  public final void signalAll() {
  }
  
  // 与signalAll类似，但它只会将等待最久的节点加入AQS锁等待队列
  public final void signal() {
  }
  
  // 释放当前线程持有的锁，将自己加入链表，然后挂起
  public final void await() throws InterruptedException {
  }
  
  // 与await类似，但它有超时时间
  // 超时时间到了还没获取到锁就会返回
  public final long awaitNanos(long nanosTimeout){
  }
}
```

接下来，具体分析源码。从`await`方法开始：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程加入等待队列
    Node node = addConditionWaiter();
    // 释放当前线程持有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 自旋式检测node是否已被转移到AQS的锁等待队列
    while (!isOnSyncQueue(node)) {
        // 挂起自己
        LockSupport.park(this);
        // 检测是否被打断过
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 代码运行到此说明node已经进入到了AQS的锁等待队列
    // 那么接下来的逻辑就是获取锁的流程了，参见3.1.2小节
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

// 根据当前线程构建条件等待节点，并加入等待队列
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 清理掉已被取消的node
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 构建条件等待节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 加入条件等待队列
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

// 释放node持有的锁
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        // 释放锁，具体代码参见3.1.3小节
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException(
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
              
final boolean isOnSyncQueue(Node node) {
    // 判断是否仍处于条件等待队列
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // 如果有前驱、也有后继节点，那么很明显已经在AQS的锁等待队列了
        return true;
    
    // 如果是其他情况，则从后往前从AQS锁等待队列查找node是否在其中
    return findNodeFromTail(node);
}
```

接下来看看`signalAll`流程：

```java
public final void signalAll() {
    // 如果当前线程没有持有锁，那么抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);  // 转移条件等待队列所有的node
}

private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    // 转移条件等待队列所有的node
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        // 转移
        transferForSignal(first);
        first = next;
    } while (first != null);
}

final boolean transferForSignal(Node node) {
    // 尝试修改node的状态
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
   
    // 进入AQS的入队流程，详情见3.1.2小节
    Node p = enq(node);
    int ws = p.waitStatus;
    // 如果node的状态是已取消，或者没有将状态变更成SIGNAL，那么尝试唤醒node的线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

