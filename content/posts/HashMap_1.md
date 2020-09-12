---
title: 【源码学习】JDK7 HashMap
date: 2018-05-30
draft: false
---

## 关键成员属性

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 默认初始大小：16

static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 扩容因子

static final Entry<?,?>[] EMPTY_TABLE = {};
// HashMap实际就是一个Entry数组

transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
// HashMap实际存储的数据类型

```

Entry 数据结构

```java
static class Entry<K,V> implements Map.Entry<K,V> {
final K key;
V value;
Entry<K,V> next;
int hash;

Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
    }
}
```

* Entry 作为静态内部类实现了 Map 接口中的内部接口 Entry
* Entry 呈链表结构，其中 next 属性为指向下一个节点的指针，而成员属性中的 table 即是由 Entry 构成的数组，也就是说 HashMap 的内部存储结构为「数组 + 链表」的形式，同时使用拉链法处理 hash 冲突
* 由 Entry 的构造方法可以看出，Entry 链表插入元素时采用头插法，这么做是因为头插法相比尾插法要快很多

## 初始化

HashMap 不会直接使用传入的值作为初始容量，而是会使用比传入值大的 2 次幂作为初始容量。而在已知存放的元素个数时，推荐使用此公式得到传入的初始容量：

$$expectedSize / 0.75F + 1.0F$$

这个公式来自 JDK8 中 HashMap 的 putAll 方法，因为此方法的作用是将 map 中的映射拷贝到新的 map 中，在拷贝时已知元素的个数，使用此公式计算新容器的初始容量更加合适，确定容量时，考虑到了扩容因子的影响，不至于过早发生扩容，减少扩容的几率，但会占用较多的内存。

## hash()

根据散列表的结构，存储时需要计算 key 的 hash 值以确定存储的位置，在 HashMap 中此过程分为两步：二次散列以及确定索引

以下为 HashMap 中的散列函数：

```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

令高位尽可能地也参与进散列值的计算，能在一定程度上改善性能较差的 hash 方法，减少碰撞。

以下方法为确定索引的方法，传入 hash 值及数据长度，返回数组的索引：

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

* 当长度为 2 次幂时，`h & (length - 1)` 与 `h % length` 等价，但是位运算更高效
* 2 次幂减 1 结果的二进制表示为全 1，在进行与散列值的与运算时，其结果即为散列值的后几位，若不是 2 次幂，位运算的结果会出现 0，影响上一步的散列结果
* 由于 hash 的结果是 int 类型，取值范围是 $[-2^{31},2^{31}-1]$，负数取模比较麻烦，而位运算可以避免这个问题：`length - 1` 一定整数，有符号数最高位是 0，与运算后结果最高位仍为 0，最终结果也一定是正数

## put()

```java
public V put(K key, V value) {

    // 若table为空，则根据阈值进行初始化
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }

    // key可为null
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    // 遍历链表，若索引位置为空则跳过循环直接在此位置插入
    // 若不为空则首先查找是否有相同的key，有则替换旧的value
    // 循环结束则表示没有相同key，执行addEntry()
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        // 若key重复则使用新的value替换旧的value
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 将新的元素插在指定索引处
    addEntry(hash, key, value, i);
    return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
    // 在数组长度大于等于阈值并且要插入的位置已经为非空时进行扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        // key为null会直接置hash值为0
        hash = (null != key) ? hash(key) : 0;
        // 由于长度变化需要重新计算索引值
        bucketIndex = indexFor(hash, table.length);
    }

    // 插入节点
    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    // 将原链表头节点记为e，令新节点存储在链表头部并指向e
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

### hash冲突判断

```java
e.hash == hash && ((k = e.key) == key || key.equals(k))
```

* 这个 if 条件的作用是判断待插入元素与当前位置上链表元素的 key 是否相同，只有相同才能用新值覆盖旧值
* 两个 key 的 hash 值相同，不足以说明 key 相同，因为 hash 是由 key 计算出来的结果，存在不同 key 具有相同 hash 值的可能性，所以需要比较key
* 先使用 == 直接比较两个 key 的内存地址，内存地址相同则 key 对象一定相同，比较结束，整个条件为真
* 若两个 key 对象的内存地址不同，再使用 equals 比较，因为存在equals 被重写的可能，equals 的结果直接决定整个条件的真假

### 扩容条件

* 触发：**已有元素个数大于阈值** && **索引位置非空**
* 假设默认大小 16，扩容因子 0.75，阈值 12 时，会出现以下两种情况：
  * 在存入第17个元素时才触发扩容，因为前 16 个元素分别占据了 16 个位置，即使元素个数大于阈值，但前 16 个元素插入时并不会产生冲突
  * 最多存储 26 个元素，此时前 11 个元素处于一个位置，形成链表，并且元素个数小于阈值；后 15 个元素分别占据其他 15 个位置，当第 27 个元素插入时才会满则扩容条件

**这种双重判断能够更好地利用内存空间，降低扩容频率**，这也是扩容因子选用 0.75 的原因，较高的值能降低空间开销，但会增加查找成本。

## resize()

```java
// 由put克制元素个数达到阈值时，新数组的长度=2*原数组长度
void resize(int newCapacity) {
    Entry[] oldTable = table;
    // 若已达到最大容量就不再扩容
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    // 创建新的Entry[]
    Entry[] newTable = new Entry[newCapacity];
    // 复制元素
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    // 新阈值=扩容因子*新的数组长度
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}


void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

### transfer()

1. for循环遍历数组，只要链表头节点不为空则创建next指针指向下一个节点
2. 重新计算头节点的索引
3. 根据索引找到新的位置
4. 将索引指向的元素拿出来令e.next指向它，再将e放在索引指向的位置，这样就完成了头插，第一次新数组的为空，所以在第二步中A作为尾节时点指向null
5. 将e再次指向next的位置，也就是原链表中的第二个节点，重复上述过程即可，假设原链表各元素扩容后仍获取到同样的索引，那么新链表是旧链表的倒序

![insert](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_insert.jpg)

## get()

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}

private V getForNullKey() {
    if (size == 0) {
        return null;
    }
    // key为null的元素固定存储在0位置，但不能保证其他key不会hash到0位置
    // 所以需要遍历此处的链表以返回key为null的value
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}

final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }

    // key为null则将hash值直接置0
    int hash = (key == null) ? 0 : hash(key);
    // 根据hash值找出索引，遍历链表
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
            e != null;
            e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

取元素时的判断逻辑与 put 类似，需要同时判断 hash 和 key 是否相同，这也是为什么**重写equals时必须重写hashCode**的原因：假设只重写 equals()，则会存在 key 对象相同（这个相同就是由 equals() 判断的）而 hash 值不同的情况，而 get 时优先判断 hash 值，会导致 equals 判断相同的 key 对象的 value 无法被容器查找到。

## 并发安全

HashMap 的扩容过程在单线程下是线程安全的，但是在多线程的情况下会发生问题，考虑以下场景：

1. HashMap 已达到扩容的临界点，线程 A、B 在同一时刻执行 put 操作后进行扩容

    ![concurrent1](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_concurrent1.jpg)

2. 此时两个线程开始执行 transfer 操作，假设B线程遍历原数组时遍历到 Entry4 时，执行完以下代码时被挂起

    ```java
    Entry<K,V> next = e.next;
    ```

    此时对线程 B 来说

    ```java
    e -> Entry4
    e.next -> Entry2
    ```

3. 假设当 A 执行完 transfer 操作后如下所示，其中 e 为 B 线程的引用

    ![concurrent2](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_concurrent2.jpg)

4. 此时线程 B 恢复执行，当执行到以下代码时

    ```java
    int i = indexFor(e.hash, newCapacity);
    ```

    显然此时对于线程 B，i=2，因为线程 A 对于 e 指向的 Entry4 索引的结果为 2。继续执行完此次 for 循环，Entry4 放入了 i=2 的位置，此时

    ```java
    e -> Entry2
    next -> Entry2
    ```

    ![concurrent3](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_concurrent3.jpg)

5. 新的一轮 for 循环，当执行到以下代码时

    ```java
    Entry<K,V> next = e.next;
    ```

    此时

    ```java
    e -> Entry2
    next -> Entry4
    ```

    继续执行完本次循环后如图所示

    ![concurrent4](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_concurrent4.jpg)

6. 再次新的一轮循环，当执行到以下代码时

    ```java
    Entry<K,V> next = e.next;
    ```

    此时

    ```java
    e -> Entry4
    next -> null
    ```

    在循环结束之前，执行到以下代码时

    ```java
    e.next = newTable[i];
    ```

    此时

    ```java
    newTable[i] -> Entry2
    e -> Entry4
    e.next -> Entry2
    ```

    再加上之前的

    ```java
    Entry2.next -> Entry4
    ```

    链法出现了环形

    ![concurrent5](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2018-05/2018_05_31_concurrent5.jpg)

    这会导致在当 get 查找一个不存在的 key 时，并且这个 key 索引的结果为 2，由于位置 2 链表出现环形，key 相等的判断条件不会出现成立的情况，造成死循环
