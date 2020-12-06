### 1. HashMap在并发情况下的死循环分析

在Java7中，在多线程环境下，使用HashMap进行put操作会引起死循环，导致CPU利用率接近100%。原因是在扩容过程中，多线程会导致HashMap的Entry链表形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry。

> sb都知道HashMap不是线程安全的，但是面试官就是会问多线程中HashMap有哪些问题。

todo:有空找找java7的源码看看。

### 2. HashMap原理

HashMap的内部实现是数组，数组元素存储的是键值对象；往HashMap中添加数据时，会按Key计算数组的索引位置，并将键值组成的Entry对象存放到该位置；如该位置已经存放了其他对象，说明发生了Hash冲突，此时会在该位置创建一个链表，并按先后顺序将Entry存放到该链表中；如果链表的长度超过8，会将链表转换成红黑树，此为性能考虑。由数组的长度是固定的，所以总有用完的时候，此时需要将数组扩充成更大的数组，实际上HashMap从效率和Hash冲突的概率等多方面因素考量并不会等到数组满了之后才扩容，其中涉及到的一个参数：装载因子，达到负载因子长度时候就开始扩容；扩容的顺序是依次将旧数组中的元素移动到新数组。

面试时按此顺序讲解：内部实现数组 -> 计算索引 -> hash冲突 -> 链表、红黑树 -> 装载因子扩容。以此将整个知识点串联起来。

### 3. 源码分析

#### 3.1 tl;dr

- 容量使用2的n次幂是为了计算高效和减少hash冲突的几率，原因：
  - 计算高效：二进制计算效率高
  - hash冲突：计算index是`hash & (length-1)`，length-1的二进制是`11111...`，每一位都能参与运算；如果不使用2的幂，那么可能有一些index永远都存储不了元素
- 拉链法过长（默认值是8）就会转换成红黑树
- 负载因此默认是0.75，如果太高虽然会节约空间，但是会花更多的时间在put和get操作上（因为更容易hash碰撞），这个和泊松分布有关系
- 扩容后元素的位置要么不变，要么变成原来的位置+cap，因为扩容前后对比，参与运算的位只有最左边多出了一位，其他的都不变

#### 3.2 私有字段

```java
// 用于存放所有的元素
transient Node<K,V>[] table;
// map元素数量
transient int size;
// map容量（capacity * loadFactor）
// 第一次扩容时大小是12，即16*0.75
int threshold;
// 装载因子，默认是0.75f
final float loadFactor;
```

#### 3.3 辅助方法

```java
// 计算元素hash
static final int hash(Object key) {
    int h;
    // 1、计算hash code
    // 2、hash高位运算
    // 3、hash和高位进行xor，其实就是高16位与低16位xor
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 计算比cap大的2的幂的那个树
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

// 计算元素的索引位置
// jdk1.7的源码，jdk1.8没有这个方法
// 因为这2个版本都是通过这种方式计算元素的索引位置
// 所以在这里也列出来了
static int indexFor(int h, int length) {
     // 这个其实就是取模运算：h%length
     // length是2^n，减1的目的就是将所有的位置为1
     // 这样再与h进行&运算，得出的结果就是余数
     // 参见SO：https://stackoverflow.com/a/10879475/1965211
     return h & (length-1);
}
```

#### 3.4 构造函数

```java
// 默认装载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

#### 3.5 添加元素

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;		// 第一次扩容（初始化）
    if ((p = tab[i = (n - 1) & hash]) == null)	// 计算索引号，并检测该索引是否已有元素
        // 如果这个位置还没有元素，直接添加
        tab[i] = newNode(hash, key, value, null);
    else {
        // 这个位置已有元素，需要做出判断：
        // key值是否相等，如果是，代表是更新元素
        // 或者 元素类型是否是TreeNode，如果是，代表已经发生过一次以上的Hash碰撞
        // 或者 就是第一次Hash碰撞
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;  // 同一个对象，更新
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 遍历该位置的元素（链表结构）
            for (int binCount = 0; ; ++binCount) {
                // 判断相同位置的元素是否有下一个元素
                if ((e = p.next) == null) {
                    // 将e添加到链表末尾
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度大于等于8，那么将结构调整为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
              
                // 判断相等性
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();	// 如果map大小超过装载阈值，那么将扩容
    afterNodeInsertion(evict);
    return null;
}
```

#### 3.6 扩容

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 旧容量，第一次是0
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧装载数，第一次是0
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               
        // 第一次初始化
        newCap = DEFAULT_INITIAL_CAPACITY;	// 默认容量16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);	// 默认装载12
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历链表，依次将所有元素添加到新的数组中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)	// 如果是红黑树结构
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 如果元素是一个链表结构，那么将链表拆分成2部分：low、high
                    // low部分插入到新数组同样索引的位置
                    // high部分插入到新数组的j + oldCap索引位置（从原来的位置移动2次幂）
                    // 之所以可以这么干，是因为扩容后(oldCap*2)，在计算索引位置时，length参与运算的二进制位比上次多出一位（见indexFor方法）
                    // 所以，索引的位置要么不变，要么从原位置移动2次幂
                    // 而在这里，根本就不需要重新计算索引位置，只需要对比最高位是否均为1就能算出索引位了
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 计算最高位是否均为1，是就放到high中，否则放到low中
                        // 比如：hash是49，容量是16，它们的二进制：
                        // 16: 0 1 0 0 0 0
                        // 31: 0 1 1 1 1 1
                        // 49: 1 1 0 0 0 1
                      
                        // 16的高位和49的相应位均为1，说明需要放到high中
                        // 而49在扩容后的索引计算方式是：49 & (32 -1)，最高位和16的最高位相同
                        // 因此只需要比较最高位就行了
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
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

### 4. 参考：

- [为什么 HashMap 的加载因子是0.75？](https://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247491794&idx=1&sn=1c25e82b48411955533a317ef47a0491&chksm=fb080a46cc7f83506f69eb0b7cd2eb32680eb5232e01c8bcbe3cf783228ce7666a1f43a11e4a&mpshare=1&scene=23&srcid&sharer_sharetime=1592616942122&sharer_shareid=8b6cce4aa7804cb52b9e5a9c08be2cf4%23rd)

