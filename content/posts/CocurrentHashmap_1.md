---
title: 【源码学习】JDK7 ConcurrentHashMap
date: 2018-06-04
draft: false
---

## 认识

HashMap 是采用「数组 + 链表」的方式进行数据的存储，而在 ConcurrentHashMap 中则划分了若干个 Segment，每个 Segment 中存储一个散列表。**ConcurrentHashMap 可以看作是一个具有二级结构的 HashMap**。

![structure](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-06/2018_06_04_structure.jpg)

JDK7 的 HashMap 在并发情况下存在安全问题，虽然 HashTable 及 Collections.synchronizedMap(hashMap) 是线程安全的，但是这两种方式都是对整个 Map 加锁，在并发量大的情况下十分影响性能。ConcurrentHashMap 引入了分段锁机制，能够分别对每个 Segment 加锁，在并发读写情况下每个 Segment 互不影响。

```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
    private static final long serialVersionUID = 2249069246763182397L;
    final float loadFactor;
    Segment(float lf) { this.loadFactor = lf; }
}
```

可以看到，Segment 继承了 ReentrantLock，它本质上就是一个可重入锁，所以这种结构相比 HashMap 能够避免并发问题。

## 关键成员属性

```java
// 整个map的默认初始容量
static final int DEFAULT_INITIAL_CAPACITY = 16;

// HashEntry数组的扩容因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 并发度，也就是segment数组的长度
static final int DEFAULT_CONCURRENCY_LEVEL = 16;

// size()中尝试获取锁的次数
static final int RETRIES_BEFORE_LOCK = 2;

// scanAndLockForPut()中尝试获取锁的次数，单核为1，多核为64
static final int MAX_SCAN_RETRIES = Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

// segment 掩码
final int segmentMask;

// segment 寻址偏移量
final int segmentShift;
```

其余属性与hashmap类似。

## 构造方法

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 根据并发等级确定数组长度，长度不可变，且2^sshift=ssize
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 根据默认值算出segmentShift=28，segmentMask=15
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    // 这里initialCapacity指整个map的初始大小
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 计算一个segment的容量
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 最小容量为2，这样根据0.75的扩容因子，插入第一个元素不会扩容，插入第二个才会扩容
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    // 创建第一个segment对象，以后创建segment均以此为原型
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                            (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 将s0放到ss[SBASE]处，即ss[0]
    // UNSAFE类内部使用CAS机制操作
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

如果使用无参的构造方法初始化，那么初始化完成后：

* Segment 数组长度为 16，**不支持扩容**
* 只对 0 位置上的 Segment 进行了初始化，其他位置为 null
* 每个 Segment 的默认大小为 2，负载因子为 1.5，所以插入第二个元素就会触发扩容
* 此时 segmentShift 为 28，segmentMask 为 15

## 存放元素

### ConcurrentHashMap 中的 put()

```java
public V put(K key, V value) {
    Segment<K,V> s;
    // 不能存null值
    if (value == null)
        throw new NullPointerException();
    // 得到key的散列值
    int hash = hash(key);
    // 仍旧是与hashmap类似，从key的散列值再得到索引
    // 不过这里得到的是segment数组的索引，可以看作是一级索引
    int j = (hash >>> segmentShift) & segmentMask;
    // 在构造方法中我们知道初始化阶段只初始化了segment[0]，其他位置为null
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
            (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // 所以当索引对应的位置内容为null时，需要创建此位置对应的segment
        s = ensureSegment(j);
    // 调用segmemt的put()进行插入操作
    return s.put(key, hash, value, false);
}
```

这个方法是 ConcurrentHashMap 直接提供的用于进行插入操作的 API，逻辑很简单：

* 通过 hash 值找到 Segment 数组的索引
* 把 put 操作交给 Segment 的 put() 进行

**关于 Segment 索引的获取**：

* hash 码为 32 位，在构造方法中得到 segmentShift = 28 / segmentMask = 15
* 将 hash 码无符号右移 28 位，也就是将高 4 位移动至低 4 位，再与 15 进行与操作

其实与 HashMap 中 `hash & (length - 1)` 而得到索引十分类似：假如不使用默认并发度 16，而改为 32，则 segmentShift = 27 / segmentMask = 31，这个 segmentShift 使得在 32 并发度下 hash 码右移后的值多了一位，而新的 segmentMask 值也刚好可以覆盖到 hash 值多了的一位。

可以认为这种计算方式可以在并发度增加时，相应地增加 hash 码实际使用的有效位数，从而令 Segment 索引分配的均匀性与 Segment 数组长度始终相匹配。

### ensureSegment()

因为初始化容器后只有 Segment[0] 存在，其他位置都是 null，当 put() 中计算出的 Segment 索引不在 0 位置时，由这个方法来创建指定位置上的 Segment：

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    // 计算getObjectVolatile用到的偏移量
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    // 第一次判断是否为null，注意这里的getObjectVolatile可以保证内存可见性
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 取到之前创建的segment
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        // 创建HashEntry数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        // 再次检查
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                    == null) {
                // 使用CAS的方式,当ss偏移u位置内存上的值为null，则替换为seg
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

因为 put() 是无锁的，所以 ensureSegment() 可能存在多个线程创建同一个位置 Segment 的情况，这里使用 CAS 来同步 Segment 的创建过程：

* 第一次 if 判断：尝试在指定索引位置取segment对象，如果为 null 则使用之前 0 位置的 Segment 对象作为镜像创建当前位置的 Segment 对象
* 第二次 if 判断：检查在创建当前位置 Segment 过程是否有其他线程已经创建过了此位置的 Segment
* while 循环：当 CAS 失败说明此位置的 Segment 已被其他线程创建，再次回到循环的判断处时就可以将 Segment 赋值给 seg，并跳出循环

### Segment 中的 put()

这个方法就是实际存放元素的核心逻辑：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 尝试获取此segment的锁
    // 获取成功则新的节点置为null
    // 否则就通过scanAndLockForPut()创建node，详见后文
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // segment内部的数组
        HashEntry<K,V>[] tab = table;
        // 获取索引
        int index = (tab.length - 1) & hash;
        // 获取HashEntry数组index处的链表的头节点指针
        HashEntry<K,V> first = entryAt(tab, index);
        // 遍历链表
        for (HashEntry<K,V> e = first;;) {
            // 若当前节点不为空
            if (e != null) {
                K k;
                // 判断是否key重复，注意这里的判断条件与hashmap中有一点不同
                // 我的理解是这里的索引已经是二级索引，冲突的可能性较低，所以没有先判断hash码
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    // 重复则覆盖旧值
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    // 插入操作结束，跳出循环
                    break;
                }
                // key不重复则继续遍历
                e = e.next;
            }
            // 若当前节点为空，说明没有重复的 key，已经遍历到链表尾；要么就是该位置没有元素
            else {
                // 如果通过scanAndLockForPut() 获得了node则将其插入链表头
                if (node != null)
                    node.setNext(first);
                // 说明获取锁成功，现在新创建node节点
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 判断是否需要进行扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    // 将node放在链表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 插入操作完成后必须释放本segment的锁
        unlock();
    }
    return oldValue;
}
```

Segment 中的 put() 有独占锁保护，所以操作并不复杂，与 HashMap 插入元素的逻辑类似：

![segment put()](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-06/2018_06_04_Segment_put%28%29.png)

需要注意的是 node 在获取锁成功时赋值 null，个人理解是如果存在重复 key 那么直接替换 value 即可，将新建节点的操作延迟到真正需要插入新节点时，以节省性能。

### scanAndLockForPut()

根据 Segment 中 put() 的逻辑可以看出，scanAndLockForPut() 的作用就是获取独占锁，并给 node 变量赋值：

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    // 循环获取锁
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            // 若头结点为null并且node为null则新建node节点
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            // 如果发现存在重复的key，则标记retries为0，等待抢锁成功退出方法即可
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        // 当尝试获取锁的次数达到一定次数后阻塞线程，直至获取锁
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        // 在「尝试偶数次」时判断当前 Segment 处链表头的节点是否是之前的值
        // 如果不是则说明有其他线程进行了插入操作（因为在 put() 中采用头插法插入节点）
        // 则重置 first 和 retries，重新进行 scanAndLockForPut()
        else if ((retries & 1) == 0 &&
                    (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

这个方法可以看作是一个自旋锁，在尝试获取锁一定次数后就会阻塞该线程直至获取锁。

需要注意的是，最终返回的 node 可能不为 null，即当前位置没有链表，则新建 node 节点并返回，这个节点在外层的 put() 中直接会被插入链表头；也可能为 null，存在两种可能导致这种情况：

* 第一次 tryLock() 就获取到锁，根本没有机会进入 while 中给 node 赋值
* 当前位置已经有链表，而且找到了重复的 key，此时会直接将 retries 置 0，代表接下来只需要等待抢到锁就可以退出方法，此时 node 也为 null，这个逻辑正好呼应了外层 put() 的逻辑：因为外层会先进行重复 key 的查找，查找不到才有新建节点，而 scanAndLockForPut() 中此时已经确定存在重复 key，新建 node 就成了无意义的操作

## get()

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    // 计算散列码，同put
    int h = hash(key);
    // 计算索引，与put类似
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 判断对应的segment与tab均不为null
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        // 遍历HashEntry数组
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                    (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                e != null; e = e.next) {
            K k;
            // key相同则取其value即可
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

get() 的逻辑十分清晰：

1. 计算 hash，找到 Segment
2. 在 Segment 中找到对应的 HashEntry
3. 遍历链表

需要注意的是，get 过程是不需要加锁的，因为 Segment 中的 table 是 volatile 的，対它的修改可以被其他线程立即可见。

另外，**ConcurrentHashMap 只能保证单次操作是线程安全的**，但是如果一个线程里进行多个操作，则无法保证线程安全：

```java
public static void main(String[] args) throws InterruptedException {
    ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<String,Integer>();
    map.put("key", 1);
    ExecutorService executorService = Executors.newFixedThreadPool(100);
    for (int i = 0; i < 1000; i++) {
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                int value = map.get("key") + 1;
                map.put("key", value);
            }
        });
    }
    System.out.println("------" + map.get("key") + "------");
    executorService.shutdown();
}
```

上述代码的执行结果不一定是 1001，因为 get 不加锁，在一个线程拿到 value 但还未加 1 时，其他线程也可能拿到这个值，导致类加不到 1001 的情况，解决的方法是使用原子类或加锁。

## size()

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    // 这里的retries变量与自旋锁方法中的类似
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // 如果尝试次数达到默认值则强制对所有segment加锁
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            // 遍历segment数组，累加segment的修改次数与内部元素的数量
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    // 如果超过int类型的最大值会变为负数
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 如果本次计数得到的总修改次数与上一次相同，则计数结束
            if (sum == last)
                break;
            // 将本次计数得到的总修改次数保存在last中以便下一次使用
            last = sum;
        }
    } finally {
        // 强制加锁并再次计数后必须释放锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

size() 是一个跨 Segment 操作，所以避免不了锁的获取，但是直接全部加锁对性能会有较大影响，所以采用了类似自旋锁的操作：

* 为了不直接锁住所有segment，认为计数过程中不会有别的线程进行修改
* 在遍历 Segment 的过程中记录 Segment 被修改的次数，然后判断本次计数中得到的修改次数是否以上一次相同
* 当尝试一定次数后发现仍有其他线程进行修改，则対所有 Segment 加锁
