---
title: 【源码学习】JDK8 ConcurrentHashMap
date: 2018-06-18
draft: false
---

## 认识

与 JDK7 相比舍弃了复杂的分段锁机制，所有的并发安全均由synchronized 与 CAS 机制保证，在存储结构上延续了 JDK8 HashMap 的「链表 + 红黑树」的结构。

## 关键属性

```java
// 最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认容量，与hashmap相同
private static final int DEFAULT_CAPACITY = 16;

// 默认并发等级
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

// 扩容因子，不再赘述
private static final float LOAD_FACTOR = 0.75f;

// 链表转化为红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

// 红黑树最小容量
static final int MIN_TREEIFY_CAPACITY = 64;

// 重要属性，扩容校验码位数，详见后文
private static int RESIZE_STAMP_BITS = 16;

// 参与扩容的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

// 扩容校验码移位数，详见后文
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

// 重要属性，表示此节点正在扩容中，hash值为-1
static final int MOVED     = -1; // hash for forwarding nodes

// node数组
transient volatile Node<K,V>[] table;

// 扩容后的数组
private transient volatile Node<K,V>[] nextTable;

// 十分重要的属性，详解后文
private transient volatile int sizeCtl;

// 重要属性，并发扩容索引，详见后文
private transient volatile int transferIndex;
```

sizeCtl 是保证并发安全最重要一个属性，被标记为 volitale，以CAS 方式来对其设值。不同状态下此属性不同的值有不同的意义，在此处先一并列出，便于后文参考：

* 初始状态：
  sizeCtl = 0；表示 table 还未被初始化
* 正常状态：
    sizeCtl > 0：即 0.75n，也就下一次扩容的阈值，其中 0.75 为扩容因子
* 初始化状态：
    sizeCtl = -1：表示 table 正在被初始化
* 扩容状态：
    **sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2 :第一个发起扩容的线程会将 sizeCtl 置为此值，也就是说此时只有一个线程在执行扩容，需要注意的是，此时 sizeCtl 是一个负值**

## 关键方法

### 校验码方法

```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

这是扩容时提供的一个生成校验码的方法，我们暂时不用考虑此方法的使用场合，只需知道这个方法做了什么即可。

这个方法的参数是n，表示此时是基于长度为n的table进行扩容而生成校验码，也就是说生成的校验码首先与本次要扩容的数组长度是有关的

* n = 16时：
    Integer.numberOfLeadingZeros(n) = 0001 1100
    resizeStamp(16) = 0001 1100 | 1000 0000 0000 0000 = 1000 0000 0001 1100

结合此方法理解sizeCtl的这个值：sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2

* n = 16时：
    sizeCtl = 1000 0000 0001 1100 << 16 + 0010 = 1000 0000 0001 1100 0000 0000 0000 0010

也就是说sizeCtl在扩容的某个状态下其高16位为本次扩容的检验码，而低16位表示正在参与扩容的线程数，**+1表示初始化，+2才表示此只有一个线程参与扩容**，+3就表示有两个线程在执行扩容，**同样地每个线程退出扩容时也会将此值-1**，所以后文中出现此判断条件时：(sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT，也表示判断当前线程是否为最后一个扩容线程，只要这个线程扩容结束，那么整个并发扩容即结束

### CAS 方法

由于jdk8中的cocurrenthashmap大量使用了CAS方法，所以在这里简要介绍一下，以compareAndSwapObject为例，其他的类似：

```java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
```

此方法有4个参数，对象var1，偏移值var2，预期值var4，要修改的值var5，如果var1偏移var2内存地址上的值和var4相等，那么就把此内存地址上的值修改为var5并且返回true，否则返回false

### 构造方法

这里只给出参数最多的一个构造方法，其他构造方法只是这个的部分逻辑

```java
public ConcurrentHashMap(int initialCapacity,
                            float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 初始容量最小与并发等级相同
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

这里需要注意的是计算sizeCtl值的过程：通过初始容量与扩容因子，取到size值，然后通过tableSizeFor(int(size)取到大于等于size整数的最小2的n次方；如果是默认扩容因子0.75那么size就是1.5倍的initialCapacity再加1。这里保证的是当初始化table时若sizeCtl>0，那么table长度一定是2的n次方

## put()

如果插入新的元素，则返回值为 null；如果覆盖了旧值，则返回旧值。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 计算hash码
    int hash = spread(key.hashCode());
    // 记录链表长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果table数组为空说明还未初始化，需要首先调用initTable()进行初始化，详见后文
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 如果table不为空则根据hash计算索引
        // (1)索引节点为空，则直接新建一个Node放在此位置即可
        // cas成功则put结束
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 注意这里仍旧使用CAS的方式给数组某位置赋值，防止其他线程已经插入后本线程重复插入
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // (2)如果此节点的hash值为MOVED则说明此位置的链表正在进行扩容
        // 本线程调用helpTransfer()进行辅助扩容，详见后文
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // (3)此索引位置已有节点
        else {
            V oldVal = null;
            synchronized (f) {
                // 防止其他线程操作，这里再次判断i位置是否仍为f节点
                if (tabAt(tab, i) == f) {
                    // 索引位置节点hash值大于0，说明此处是一个链表
                    if (fh >= 0) {
                        // 记录链表长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 有重复key则覆盖旧值，put结束
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 若无重复key则将新建节点插到链表末端，put结束
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    // 如果索引节点是红黑树节点则调用红黑树的put方法
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 如果binCount != 0说明向链表或红黑树中添加一个节点成功
            if (binCount != 0) {
                // 若链表长度大于转为红黑树的阈值则进行转换操作，详见后文
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 将当前concurrenthashmap元素数量+1并判断是否需要扩容，详见后文
    addCount(1L, binCount);
    return null;
}
```

整个 put() 的过程抛开细节看逻辑还是较为简单的：

![put](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-06/2018_06_04_jdk8_put%28%29.png)

### initTable()

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 判断table是否未初始化
    while ((tab = table) == null || tab.length == 0) {
        // 如果sizeCtl为-1则说明其他线程正在执行初始化，线程让出资源
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 进行初始化，使用cas方法将sizeCtl置为-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 再次判断，第一个线程初始化好后，循环中的其它线程到此时不用再初始化
                if ((tab = table) == null || tab.length == 0) {
                    // 如果选用的构造方法已经对sizeCtl赋过值，那么就取这个值为数组长度
                    // 否则sizeCtl为0，使用默认数组长度
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 将此时table长度的0.75倍作为下一次扩容的阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                // 将sc赋给sizeCtl，此时sizeCtl处于正常状态，即0.75n
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

整个initTable()就是保证初始化工作只由一个线程进行，因为put可以并发执行，而initTable()需要确保单线程执行

### treeifyBin()

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // 如果链表长度超过转换为红黑树的阈值但是table长度并没有超过64
        // 则进行扩容操作，详见后文
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        // 否则将index位置的链表转换为红黑树
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 加锁
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 遍历链表
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                                null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

这个方法转换红黑树的操作并不是我们重点关心的，主要是为了引出这个方法里导致扩容的场景

至此在put过程中与没有直接使用扩容方法的逻辑我们已经基本梳理清楚，以下开始进行扩容相关逻辑的学习

## 扩容机制

### 认识

jdk8中的concurrentahashmap的扩容过程是可以并发进行的，由于抛弃了各种锁转而使用CAS与synchronized维护线程安全，导致整个扩容过程实现起来十分复杂，但是在设计角度讲其实并不难理解，看完整个过程读者应该会和我一样惊叹于Doug Lea对整个扩容机制在并发下的精妙设计

### 扩容场合

#### addCount()

```java
// check就是在put的最后一步增加元素个数时传入的链表长度
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 使用CAS更新baseCount失败的话才进入内部逻辑
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        // 计数
        s = sumCount();
    }
    // put过程中如果新增了节点就需要在判断是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 当计数得到的总节点数大于等于扩容阈值时进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            // 计算扩容检验码，详见前文
            int rs = resizeStamp(n);
            // sc<0说明有线程在执行扩容操作
            if (sc < 0) {
                // 判断扩容是否结束，详见后文
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    // 满足以上任意条件则说明扩容结束，直接跳出
                    break;
                // 若扩容过程没有结束，则使用CAS方法将sizeCTL+1，表示当前参与扩容的线程数+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    // 扩容方法，注意这里传入的nt就是全局的nextTable属性
                    // nextTable就是扩容后的数组，tranfer过程中需要将tab的数据迁移到nextTable中，这里传入此数组令本线程也参与到数据迁移过程中
                    transfer(tab, nt);
            }
            // 如果sizeCtl大于等于0，说明当前线程是第一个发起扩容的线程
            // nextTable为null
            // 这里将sc替换为(rs << RESIZE_STAMP_SHIFT) + 2)正说明了此时本线程第一个进行扩容的线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

这个方法在put新增元素完成后执行，如果元素数量+1后满足一定条件则需要进行扩容，是触发扩容的第一种情况

此方法前半部分就是更新整个map的总的元素数量，后半部分才是用来判断是否需要扩容，方法中用来判断扩容是否结束的五个条件会在后文详细讨论

#### tryPresize()

```java
private final void tryPresize(int size) {
    // 这里c的取值与构造方法中相同
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    // 通过sizeCtl判断是否有其他线程在进行扩容
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 扩容之前需要保证table已经被初始化过
        // 这个if的逻辑与initTable()中的相同，这里不再赘述
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        // 未达到扩容阈值或以达最大容量则直接跳出
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        // 进行扩容操作，与addCount()相同，不再赘述
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

这个方法是在当索引处的链表长度超过转换为红黑树的阈值但table的长度小于MIN_TREEIFY_CAPACITY时执行，是触发扩容的第二种情况。扩容部分的逻辑与addCount()相同

#### helpTransfer()

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 判断扩容是否结束，详见后文
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            // 仍旧与以上两个方法中的if判断条件类似
            // 但是注意这里少了一个条件：(nt = nextTable) == null
            // 因为nextTable为null时说明扩容完毕了，这里先给出这个结论，详见后文
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

当进行put时索引处的节点的hash值为-1的话，说明有线程正在对table扩容，此时会调用此方法进行辅助扩容，这是会触发扩容的第三种情况，需要注意的是ForwardingNode这个类：

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    // 详见get()
    find()
}
```

这个类会持有一个nextTable的引用，指向扩容后的table数组。在扩容操作中需要将数据从table中转移至nextTable中，如果table中某个节点的数据已经被转移完毕，则在table的此位置放置一个ForwardingNode节点，其hash值为MOVED，说明扩容过程尚未完成

### 扩容过程

到此我们已经梳理了会触发扩容的三种情况，这三种情况种均含有逻辑类似的代码段，在本节我们会看到完整的扩容过程，结合扩容过程会将此之前一些或略过的代码逻辑讨论清楚，理解了扩容过程之后再看之前的触发扩容的三种场景的逻辑会更加清晰

#### 认识

这里再重申一遍，jdk8的concurrenthashmap的扩容过程是能够多线程参与执行的

由于扩容过程的实现比较复杂，这里先简要说明下扩容的大致逻辑：

1. table数组长度为n，需要将这n个位置的数据均迁移到nextTable中，所以共有n个迁移任务。每个线程负责一个迁移任务，每当完成一个迁移任务后再检查是否还有需要迁移的任务没有线程负责，有的话允许其他线程进行辅助迁移即可
2. 为了提升效率，Doug Lea大神在迁移过程中加入了一个stride变量，这个变量指示了每个线程负责的迁移任务数
3. 同时为了协调多个线程的并发迁移，使用了transferIndex变量。在迁移开始时此变量指向table的末端，**从后向前**地每stride个任务属于一个线程，然后移动transferIndex指向新的位置，重复上述过程

#### transfer()

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 选取stride的值，单核CPU为n，多核CPU为 (n>>>3)/NCPU，最小值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 迁移开始之前需要保证nextTable不为null
    // 触发扩容的三种情况会保证第一个发起扩容的线程出入的nextTab为null
    // 这里初始化后会将nextTab赋给全局的nextTable，保证后续参与扩容的线程拿到的nextTable正是正在参与扩容的nextTable
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            // 容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 将初始化完成的nextTab赋给全局的nextTable
        nextTable = nextTab;
        // 将transferIndex指向table的末端
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 生成ForwardingNode节点，作用前文已有讨论，注意这里传入了初始化后的nextTab
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // advance可以理解为一个标记，为true表示一个迁移任务已经完成
    boolean advance = true;
    // finishing为true时表示整个数据迁移过程已经完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    // i为索引，通过--i来遍历table[i]，指向当前处理的某个迁移任务
    // bound为边界索引，指向当前迁移任务段的最后一个任务
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 这个while循环就是为了进行从table[i]到table[bound]节点的遍历
        while (advance) {
            int nextIndex, nextBound;
            // 线程每次处理完一个节点后都会将advance再次置为true，使其能够再次到达这里执行--i从而进行下一个节点的数据迁移
            // 当处理完bound节点后就不会在进入此分支，会再次进入第三个分支被分配任务执行数据迁移工作
            // 当线程进入循环时时一般会先进入第三个分支领取迁移任务
            // 除非在第二个分支判断扩容结束，i=-1
            // 或finishing为true，即整个扩容过程已完成，此时同样i=0-1=-1
            if (--i >= bound || finishing)
                advance = false;
            // 这个分支是结束transfer()的条件，每个执行扩容的线程最多进入一次这个分支，因为一旦某个线程执行迁移够快以至于将最后一段也执行了，那么就会进入这个分支，
            // 当transferIndex<=0时表示已经没有需要迁移的任务了，将i置为-1，advance值为false，准备退出transfer()
            // 这里=0时表示移动到了table的头部，说明本线程执行得够快，已经将最后一段迁移任务执行完了
            // 而<0我猜测是由于并发导致多个线程重复操作所造成的，仅供参考
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 设置transferIndex，bound，i
            // 由于是从后向前遍历，所以nextIndex表示还有多少个迁移任务需要执行
            // 注意这个值在第一次迁移时取的是数组长度，也就是说nextIndex-1才指向的是table的最后一个节点
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                // i指向当前迁移任务段的第一个任务
                i = nextIndex - 1;
                // 本线程迁移任务分配完成，跳出循环
                advance = false;
            }
        }
        // 当i<0时说明本线程的扩容任务已完成
        // 至于i>=n与i+n>=nextn，这里的相关逻辑我没与看出i会满足这两个条件，猜测是仍旧是由于并发下多个线程重复操作造成的，仅供参考
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 如果整个扩容过程都完成了那么就将全局的nextTable置null
            if (finishing) {
                nextTable = null;
                // 将扩容用的临时nextTab的值赋回给全局table
                table = nextTab;
                // 计算新的扩容阈值
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 如果本线程的扩容任务完执行的够快已经将最后一段迁移任务执行完毕就会进入这个分支
            // 此处使用CAS将sizeCtl的值减1，表示有一个线程退出扩容，详见检验方法
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 判断是本线程是否为最后一个扩容线程，不是则直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 如果本线程是最后一个扩容线程就将finishing置为true
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果节点是null，那么使用CAS将ForwardingNode插入到此节点表示此节点的迁移任务已完成
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果hash值为-1说明此节点就是一个ForwardingNode节点
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // 否则就对此节点加锁，进行数据迁移操作
        else {
            synchronized (f) {
                // 判断i位置节点没有变化
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 如果节点hash值大于0说明是链表
                    if (fh >= 0) {
                        // 这里与jdk8的hashmap的rehash原理相同，不再赘述
                        int runBit = fh & n;
                        // 注意这个指针
                        Node<K,V> lastRun = f;
                        // 遍历链表
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            // 注意这里，这个分支在整个遍历链表的过程中会令lastRun指向最后一个与头节点runBit值相同的节点的下一个节点
                            // 也就是说lastRun及其之后的节点的hash&n的值与头节点均不同，这样可以一次将这些节点挂在hn指针上
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 以下与jdk8的hashmap的rehash原理相同，不再赘述
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 红黑树的迁移，不做详解,原理类似
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

1. transfer()过程可以分为两部分，前半部分就是给线程分配迁移任务，即控制advance属性来进入while循环来进行迁移任务的更新、再领迁移任务、退出等逻辑；后半部分就是线程针对一个节点执行迁移任务，两部分套在一个大的死循环里

2. transfer()只是进行数据迁移，并不是扩容的全部，这个方法需要有外围的三个会触发扩容场景的方法来控制传入此方法的nextTab是否为null

3. 关于((nextIndex = transferIndex) <= 0)分支：
    * 每个线程至多进入一次此分支
    * 假入本线程执行迁移任务够快，以至于将最后一段扩容任务都执行完毕了，那么就会进入此分支，从而进入(i < 0 || i >= n || i + n >= nextn)分支
    * 此时假设finishing为false，也就是说还有其他线程正在执行迁移任务
        * 那么本线程尝试将sizeCtl-1，-1成功则判断自己是否为最后一个扩容线程
            * 是最后一个线程的话就将finishing值为true，这样其他线程进入扩容时不会再进入((nextIndex = transferIndex) <= 0)分支，会在上一层判断finishing为true直接退出transfer()
            * 否则直接退出
        * 如果尝试-1失败则会再次进入(i < 0 || i >= n || i + n >= nextn)分支判断
            * finishing为false则重复上述过程
            * 此时finishing不会为true，因为即使第一次判断finishing为false时的那些未完成迁移的线程此时已经完成了，但是由于本线程sizeCtl-1未成功，其他线程判断(sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)时一定是成立的，那么这些线程就直接退出，没有机会将finishing置为true

4. 关于前文略过的if分支：

    ```java
    ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
    ```

    以上的五个条件均是在触发扩容时用来判断扩容是否结束
    * (sc >>> RESIZE_STAMP_SHIFT) != rs：由于校验码是基于当前table生成的，所以校验码不同说明*table长度已经变化了
    * sc == rs + 1：在校验码方法中已经讨论过+1情况是初始情况，表示没有线程参与扩容
    * sc == rs + MAX_RESIZERS：MAX_RESIZERS是能参与扩容的最大线程数
    * (nt = nextTable) == null：在transfer()我们已经知道nextTable为null表示扩容结束
    * transferIndex <= 0：同上

## get()

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    // 判断table及节点非空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断hash
        if ((eh = e.hash) == h) {
            // 判断key
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 头节点hash值小于0说明是ForwardingNode节点或红黑树节点，调用相应的方法
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get方法实在是非常简单，只有一点需要注意，当节点是ForwardingNode时，ForwardingNode重写了find方法：

```java
Node<K,V> find(int h, Object k) {
    // loop to avoid arbitrarily deep recursion on forwarding nodes
    outer: for (Node<K,V>[] tab = nextTable;;) {
        Node<K,V> e; int n;
        if (k == null || tab == null || (n = tab.length) == 0 ||
            (e = tabAt(tab, (n - 1) & h)) == null)
            return null;
        for (;;) {
            int eh; K ek;
            if ((eh = e.hash) == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
            if (eh < 0) {
                if (e instanceof ForwardingNode) {
                    tab = ((ForwardingNode<K,V>)e).nextTable;
                    continue outer;
                }
                else
                    return e.find(h, k);
            }
            if ((e = e.next) == null)
                return null;
        }
    }
}
```

find方法的逻辑并不复杂，我们需要知道的是正是由于在扩容过程中ForwardingNode的nextTable指向了全局的nextTable，所以即使在扩容过程中get方法也可以通过ForwardingNode的find方法找到处于要查找的kv对
