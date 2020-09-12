---
title: 【源码学习】JDK8 HashMap
date: 2018-06-11
draft: false
---

## 认识

JDK8 中的 HashMap 相比 JDK7 中的改进主要集中于三个方面：

* 链表长度在达到 8 时会转变为红黑树
* 対扩容后 rehash 过程进行了优化
* hash 冲突时原本使用的头插法改为尾插法

## hash()

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

相较于 JDK7 中的 hash 算法略有简化，优化了高位算法，提升了计算速度，尽量保证让散列码的高低位都参与运算。使用异或比使用与和或能使结果更均匀，与的结果会向 0 靠拢，而或的结果会向 1 靠拢

## put()

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次put操作时会触发下面的初始化操作
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 如果索引处的节点为空，则直接新建一个节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果发生hash冲突，则根据节点具体情况进行判断
    else {
        Node<K,V> e; K k;
        // (1)判断索引处节点是否key重复，重复则将这个节点赋给e供后续使用
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // (2)当这个节点是红黑树节点时就调用红黑树的put方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // (3)此时说明此索引位置存在一个链表，需要进行遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                // 如果此节点为最后一个节点，则将新的元素插在链表末尾
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // TREEIFY_THRESHOLD默认为8
                    // 也就是说新插入的元素是第8个时会触发链表转换为红黑树的操作
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在链表中查找到相同的key则直接用此节点覆盖链表的第一个节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e不为空说明发现了具有重复key的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent为true时表示不存在传入的key时才进行put
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 当插入新元素后使数组达到阈值，则需要进行扩容
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

* 链表节点数大于 8 时会转换为红黑树，因为 JDK 的开发者根据统计认为链表长度达到 8 的概率十分小，而红黑树使用的内存空间较链表更多，插入更复杂，所以需要控制转换为红黑树的频率
* 红黑数节点个数小于等于6时，又会转化回链表结构，中间的差值7用来防止链表和树频繁转换
* hash 冲突时，只有当 HashMap 元素个数大于 64 时才会将对应的链表转换为红黑树
* JDK7 中扩容时使用头插法将元素插入链表，而 JDK8 中改为尾插法，这样做能够避免在多线程下触发扩容时链表形成环形

## resize()

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // (1)普通扩容，即put操作一定次数后的扩容
    if (oldCap > 0) {
        // 超出最大值就不再扩容了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容为原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 阈值相应扩容为原来阈值的两倍
            newThr = oldThr << 1; // double threshold
    }
    // (2)当使用public HashMap(int initialCapacity)初始化时会根据传入的初始容量计算阈值
    // 在第一次put时会进入此分支将当前阈值作为数组容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // (3)当使用public HashMap()初始化时未指定阈值，只指定了负载因子为默认值
    // 此时oldThr与oldCap均为0，会进入此分支
    // 所以此处需要计算数组容量与阈值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果以上逻辑未确定新的阈值则在此计算新的阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        // 用新的数组长度初始化数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 如果是由于第一次put操作进入此方法，则到此为止，以下为数据迁移逻辑
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果不存在下一个节点直接重新计算这个节点在新数组中的索引即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果此节点是红黑树根节点则调用相应逻辑
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 否则此索引位置为一条链表
                else { // preserve order、
                    // 这里使用两条链表进行数据迁移
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

resize 的前半段可以认为是对新数组的初始化过程，以下两种情况都会执行前半段逻辑：

* 第一次 put 时发现 table 为 null，需要使用 resize 对 table 进行初始化
* 触发扩容时

### 数据迁移过程

JDK8 的 Hashmap 在数据迁移时对 rehash 过程做了优化，由于扩容使用的数组长度为 2 次幂，在计算索引位置时由于 n 变为之前的 2 倍，那么 n-1 的二进制表示与扩容前相比仅仅是在高位方向多了一位 1，也就是说扩容后计算索引时hash值会多算进去一位

* 如果 hash 值多算进去的那一位是 0，那么扩容后元素的索引不变，因为 n-1 的二进制为全1，与 n-1 相与的结果不变
* 如果是 1，新的索引位置就是旧的索引位置加上旧的数组长度

![rehash](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-06/2018_06_11_rehash.jpg)

因此只需判断 hash 值多被计算进去的一位是 0 还是 1 即可，也就是代码中的`(e.hash & oldCap) == 0`，注意这里是 oldCap 而不是 oldCap - 1 。

另外就是由于临时链表使用了头尾双指针，若旧数组索引相同的元素在新数组中索引位置仍相同，新数组中的链表不会倒置。
