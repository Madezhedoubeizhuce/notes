## 存储结构

HashMap使用链式存储，table存Node节点，Node是单链表，当Node节点存储的元素数量大于等于8时，会将其转换为红黑树

## ![image-20210320180055405](.\imgs\image-20210320180055405.png)初始化

- table容量为2^n，默认为16，指定容量，会通过tableSizeFor()调整为最接近的2^n，至于为什么要是2^n稍后再分析

```java

    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

	static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

- threshold为table容量乘以loadFactor，loadFactor默认为0.75（设置过大，导致增加查询时间，过小，浪费内存空间）

## 存取元素

put或get元素时，通过`(n - 1) & hash`获取table下标

为什么要使用`(n - 1) & hash`来计算table的下标呢？

1. 如果直接使用hash值作为数组下标，需要容量很大的数组，因此一般需要基于table值映射到table的某个位置，最直接的方式当然就是求余
2. 前文提过，table的容量都是2^n，而`(n - 1) & hash`和直接求余的结果恰好是相同的，而且求模运算的效率远低于位运算，因此这里使用`(n - 1) & hash`进行求余

下面是put元素的实现方法，在注释TAG1处可以看到是通过`(n - 1) & hash`定位table下标的

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // TAG1: 根据hash值定位table位置
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
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
        resize();
    afterNodeInsertion(evict);
    return null;
}
```



	## 扩容

​	size超过threshold时，使用resize进行扩容，table容量扩为两倍，循环遍历旧数组（下标j），使用`(e.hash & oldCap) == 0`将旧数组的链表拆分为两个链表，分别将两个链表存在j和j+oldCap的位置

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // TAG1: 扩容
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
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
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
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // TAG2：分割链表，将(e.hash & oldCap) == 0的节点保留在原位置
                    // 其他节点保存在j + oldCap的位置
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
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

首先，看到TAG1注释处，这里把新的容量和阈值都左移一位，也就是将容量扩大了两倍。

接着看到TAG2注释处，这里处于旧table的for循环内，这个else分支表示当前位置存储着一个链表，在这里对此链表进行遍历，通过`(e.hash & oldCap) == 0`这个条件将其中的每个节点分割到hiHead和loHead两个链表中，并把hiHead保存在newTab[j + oldCap]中，把loHead保存在newTab[j]中，这样的操作有什么意义呢？下面就分析一下：

首先，我们要了解`(e.hash & oldCap) == 0`这句代码的含义：在前面讲过，table的容量都是2^n，并且存取元素时都是通过`(n - 1) & hash`来定位key对应的table位置。

- 假设这里table是容量为16（oldCap），也就是n为0x10000，n-1就是0x01111，假设hash值为0x10111，那么`(n - 1) & hash`的值就是0x00111，即index为7。现在table调用resize扩容了，也就是n变成了0x100000，n-1为0x011111，重新计算后的下标就是0x10111，即新的index为23，也就是7+16（oldCap）。如果hash值为0x00111，扩容前后的index都是7。
- 到这里再思考一下`(e.hash & oldCap) == 0`这个条件，其中oldCap是2^n，那么就是判断hash值的第n位是否为0，如果为0，则说明扩容前后此节点所对应的index没有变化，则将其放到loHead链表中以保存在原来的位置中，如果hash值的第n位为1，则说明扩容后对应节点的下标为原始下标+2^n，也就是j+oldCap。

总的说来，就是扩容后，table的容量改变之后，导致每个节点在table中的位置都需要同步更改，因此这里使用这种方式来更新节点所处的位置。

## 为什么table容量需要是2^n

- 在存取元素中讲过，需要将hash值映射到table的位置，通常都是hash%table.length，table容量为2^n可以使用位运算替代求模运算，提高效率
- table容量为2^n可以减少hash映射到table的位置冲突，因为它是将hash值的n-1位都保留下来
- 若table容量不为2^n，在求余时可能会破坏hash的原始分布

## HashTable

HashTable内部使用(hash & 0x7FFFFFFF) % tab.length获取桶下标，也是用链式法解决hash冲突，且每个方法都为同步方，以确保线程安全。