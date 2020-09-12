---
title: 【源码学习】AQS 独占模式
date: 2019-09-30
draft: false
---

## 认识

AbstractQueuedSynchronizer 是一个抽象类，它封装了各种同步机制的底层细节，显式锁 ReentrantLock 就是基于 AQS 实现的。它也可以用来定义自己的同步工具，只需要继承这个类并重写一些它的方法即可。

synchronized 是被封装在 JVM 中的，与操作系统相关，并非由 JDK 实现，而 AQS 则完全是由 Java 代码实现，比其他使用系统底层的实现不知道高到哪里去了。

## 独占模式 & 共享模式

AQS 中维护了一个 state 属性，由 volatile 修饰，用来表示线程的同步状态，关于这个属性的几个方法都是 protected 和 final 的，也就是说只能在子类中使用而且不允许被重写。通过设置 state 属性来实现多线程的独占模式或共享模式：

独占模式：

* state 初始值为 0
* 线程需要进行独占操作前判断 state：
  * 如果是 0，则将 state 置 1，该线程进行操作。先判断再操作的过程使用的就是 CAS 机制
  * 如果是 1，就表示其他线程正在进行独占操作，本线程阻塞
* 获取独占状态成功的线程执行完毕后需要释放同步状态，将 state 置 0

共享模式：

* 与独占模式类似，state 的值表示允许存在这么多个线程同时执行
* 每一个进入共享模式的线程将 state 减 1
* 如果 state 的值不大于 0 则该线程需要阻塞等待
* 释放同步状态则将 state 加 1

## JDK 下竞争同步状态的实现

基于同步状态，就可以将「获取同步状态」和「释放同步状态」这两种操作进行抽取，以这两种操作为基础，就可以定义各种并发工具。

AQS 提供了几个方法来供我们自定义获取或释放同步状态的方式，它们都需要在子类中进行具体的实现：

* tryAcqure：独占模式下获取同步状态，获取成功则返回 true
* tryRelease：独占模式下释放同步状态，释放成功则返回 true
* isHeldExclusive：独占模式下当前线程是否已经获取同步状态，在使用到 Condition 时才需要实现它

下边是一个使用 AQS 的示例：

```java
public class MySync extends AbstractQueuedSynchronizer{

    @Override
    protected boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);
    }

    @Override
    protected boolean tryRelease(int arg) {
        setState(0);
        return true;
    }

    @Override
    protected boolean isHeldExclusively() {
        return getState() == 1;
    }
}
```

在这个示例中，只要将 state 从 0 变为 1，就认为成功获得了独占的同步状态，対是否处于同步状态的判断也只是通过判断 state 的值。获取和释放同步状态根据不同场景来具体实现。

也就是说**具体到我们自定义的同步器，它能不能重入、是否公平，都是由我们自行在 AQS 提供的这几个模板方法中实现**。

需要注意的是，这些方法都由 protected 修饰，也就是说它们只能在我们的自定义工具类中被重写，而其他逻辑组成完整的操作同步状态、供我们直接调用的 public 方法，则是由 AQS 提供的**模板方法**。

另外，AQS 继承了 AbstractOwnableSynchronizer，具有这个属性：

```java
private transient Thread exclusiveOwnerThread;

protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
}

protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
}
```

通过这个属性和相关方法，在各种 AQS 的子类中可以记录独占模式下成功获取同步状态的线程。

### 同步队列

AQS 维护了一个队列，这个队列用来存储获取同步状态失败的线程，以及通知处于这个队列中的线程再次去获取同步状态。**AQS 借助它实现了 JDK 层面的线程阻塞和恢复**。

```java
static final class Node {

    volatile Node prev;
    volatile Node next;
    volatile Thread thread;

    volatile int waitStatus;

    // waitStatus 的值
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    // 以下两个 waitStatus 的值在独占模式中也不会用到
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    // 独占模式下不会用到此属性
    Node nextWaiter;
}
```

```java
private transient volatile Node head;
private transient volatile Node tail;
```

可以看到基本的属性包括前后指针和保存的内容：Thread，而 AQS 维护了队首和队尾指针。

### 独占模式下获取同步状态

#### acquire()

acquire() 就是由 AQS 提供的、调用我们自行实现的 tryAcquire() 的模板方法，它还包含了线程获取同步状态失败的的阻塞逻辑。

```java
public final void acquire(int arg) {

        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```

这个方法的逻辑十分简单：

* 如果 tryAcquire() 返回 true，也就是说根据我们自定义的方式线程获取了同步状态，那么就直接返回
* 否则使当前线程阻塞，阻塞操作就是由 acquireQueued() 和 addWaiter() 完成的
* selfInterrupt() 在文末进行解释，不影响主要的阻塞逻辑

#### addWaiter()

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 如果尾指针不为空，则将新的节点插入队尾
    if (pred != null) {
        // 更新前向指针
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            // 更新后向指针
            pred.next = node;
            // 返回新插入的节点
            return node;
        }
    }
    // 否则新建队列対插入节点
    enq(node);
    // 返回新插入的节点
    return node;
}
```

addWaiter() 实现的就是向同步队列中插入节点，如果 tail 指针为空，说明队列还未创建，则使用 enq() 新建队列対插入节点：

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // tail 为空则直接创建一个节点
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 更新前向指针
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                // 更新后向指针
                t.next = node;
                return t;
            }
        }
    }
}
```

新建队列并插入节点后的队列如图所示：

![new node](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2019-09/2019_09_30_enq.jpg)

需要注意的是，**head 节点在队列中仅仅标记队首，不会存储线程对象**，waitStatus 初始化为 0。

#### acquireQueued()

执行到 acquireQueued() 就说明 tryAcquire() 获取同步状态失败，并且将当前线程作为一个节点通过 addWaiter() 加入到了同步队列中。

不论是否进行新建队列的操作，addWaiter() 都会返回新插入的节点，与 enq() 的返回值无关。这个节点会被当作 acquireQueued() 的参数之一

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 判断当前节点的前驱节点是否为 head，如果是就再次尝试获取同步状态
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

acquireQueued() 中会先判断当前节点的前驱节点是否为 head 节点，如果是就再次尝试获取同步状态，因为 head 节点仅作为标记，如果前驱节点是 head，那么当前节点就是就是队列中的第一个存有线程对象的节点，如果之前成功获取同步状态的线程此时释放了同步状态，那么此时的 tryAcquire() 就可以「侥幸」地获得同步状态。

获得同步状态后会把当前节点变为 head 节点：

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

可以看到 head 指向当前节点后，让当前节点的 prev 指针断开与前驱节点的连接，同时 p.next = null 断开了前驱节点与当前节点的连接，也就是说丢弃了原本的 head，令当前节点成为了新的 head 节点，是一个**变相的出队操作**。

需要注意的是 setHead() 及第一个 if 内的逻辑都没有使用 CAS，因为执行这些逻辑的前提是 tryAcquire() 已经成功了，相当于是在已经获取同步状态下进行的操作。

#### shouldParkAfterFailedAcquire()

如果之前在 acquireQueued 中尝试再次获取同步状态时失败，就会执行到这个方法，它会根据前驱节点的 waitStatus 来决定是否阻塞当前线程：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱节点的 waitStatus 属性
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

可以看到首先获取了前驱节点的 waitStatus 属性，独占模式下只会用到两个值：

* CANCELLED（值为 1）：节点代表的线程取消了排队，放弃了同步状态的竞争，会被踢出队列
* SIGNAL（值为 -1）：供后继节点使用。如果一个节点代表的线程将要被阻塞，处于等待状态，那么它前驱节点的 waitStatus 必须为 SIGNAL。换句话说，**前边节点代表的线程将要出队时，SIGNAL 表示在出队时还需要唤醒它之后的节点**。

```java
if (ws == Node.SIGNAL)
    return true;
```

如果前驱节点已经设置了 SIGNAL 状态，就表示前驱节点在出队时会唤醒当前节点，所以返回 true，表示当前节点可以安心地被阻塞，从而继续执行 parkAndCheckInterrupt() 令线程阻塞。

```java
if (ws > 0) {
    do {
        // 向前移动 prev 指针
        node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    // 将找到的 SIGNAL 节点的 next 指针指向当前节点
    pred.next = node;
}
```

如果前驱节点的 waitStatus 大于 0，也就是说处于 CANCELLED 状态，那就再向前寻找，直到找到一个节点为 SIGNAL，然后将当前节点直接插队到这个节点后边。

```java
else {
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
}
```

如果前驱节点既不是 SIGNAL 也不是 CANCELLED，就直接把它设置为 SIGNAL。这个情况是对应 Condition 的，这里不再展开。

只有当前驱节点为 SIGNAL 时才会返回 true，否则就是在 acquireQueued() 中再次循环，因为可能前驱节点变为 head 节点，所以会再次尝试获取同步状态，如果获取失败再次进入 shouldParkAfterFailedAcquire()。

也就是说 **acquireQueued() 中进行了一个自旋操作，不断地将当前节点「插队」在 waitStauts <=0（非 CANCELLED） 的节点后，直到成功获取同步状态，退出自旋**：

![acquire()](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2019-09/2019_09_30_acquire%28%29.png)

另外，finally 块中的代码其实是不起作用的，它是为响应中断式竞争同步状态服务的。

#### parkAndCheckInterrupt()

当 shouldParkAfterFailedAcquire() 返回 true 时就可以将线程阻塞了：

```java
private final boolean parkAndCheckInterrupt() {
    // 调用 native 方法将线程阻塞
    LockSupport.park(this);
    return Thread.interrupted();
}
```

需要注意的是线程会阻塞在 park() 这一行，不会向下执行，也就是说之前的 acquireQueued() 也会阻塞在 if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())。

### 独占模式下释放同步状态

#### release()

同 acquire()，release() 也是 AQS 提供的调用 tryRelease() 的模板方法：

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

unparkSuccessor() 用来唤醒后继节点代表的线程，传入的参数是 waitStatus 不为 0 的 head 节点，因为在根据只有为 SIGNAL 状态的节点才需要唤醒后继节点，head 的 waitStatus 为 0 说明没有需要唤醒的线程了（为 0 的情况出现在 addWaiter() 插入节点需要新建队列时）。

#### unparkSuccessor()

```java
private void unparkSuccessor(Node node) {

    int ws = node.waitStatus;
    if (ws < 0)
        // 这个方法中 head 节点没有唤醒后继节点的必要，所以状态清零了
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        // node.next 如果不符合条件就释放对它的引用
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 対符合条件的节点，释放其持有的同步状态
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

其中：

```java
if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
            s = t;
            // 没有跳出循环，会找到距离 head 最近的
}
```

如果 head 的后继节点不存在或处于 CANCELLED 状态，则从队尾开始遍历，寻找距离 head 最近的、waitStatus < 0 的节点（在独占模式中 waitStatus < 0 就是 SIGNAL 状态）并唤醒它。

需要注意的是：

* 从尾至头遍历节点：因为 addWaiter() 中节点的插入不是一个原子操作，有可能存在这种情况：「前驱节点的 next 指针还没有指向新插入的节点时就触发了 unparkSuccessor()」，但是此时新插入节点的 prev 指针已经指向前驱节点，所以从后向前遍历一定可以遍历到所有节点
* s == null：这个判断也是需要的，因为 head 节点也有可能因为并发的原因，导致其 next 指针没有指向后继节点，此时 s 就会为 null
* if (t.waitStatus <= 0)：这里需要等于 0 的判断，因为如果仅仅完成了一次 enq()，队列中除了 head 外只有一个节点，此时节点的 waitStatus 为 0
* s.waitStatus > 0：同理，这里也包括了上述情况

#### 中断的响应

释放同步状态时其实涉及到了两个线程：

* 一个是当前释放了同步状态的线程，并且调用了 LockSupport.unpark()
* 另一个线程就是之前阻塞在 parkAndCheckInterrupt() 中、现在被 LockSupport.unpark() 唤醒而继续执行的线程

线程在被唤醒在 parkAndCheckInterrupt() 继续执行时，会执行返回：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    // 线程被唤醒，执行 return
    return Thread.interrupted();
}
```

interrupted() 会返回当前线程的中断状态并清除中断状态。接着跳转到外层方法 acquireQueued() 中：

```java
if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
    interrupted = true;
```

如果 parkAndCheckInterrupt() 返回 true，那么 interrupted 就会被置为 true，表示在阻塞过程中有其他线程想中断当前线程，但是之前没有响应中断，所以这里记录一下。然后继续自旋操作，在 acquireQueued() 继续抢锁或再次阻塞，直到抢到锁，退出自旋时就会返回阻塞过程中的中断状态，以此在 acquire() 中判断是否需要将未响应的中断进行响应。
