# 一、源码

##  putVal(int, K, V, boolean,boolean ):V

```java
    /**
     * Implements Map.put and related methods.
     * 插入元素
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //1.如果当前table为空或长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //2.根据key的hash找到在数组中的下标，判断该下标的值是否为空
    //2.1如果为空，该位置新增该key的节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
     //2.2如果不为空，说明此位置已经有node了，产生了hash冲突
        Node<K,V> e; K k;
        //3.如果该位置的node的hash和key都和当前待插入的相同，则覆盖原来的key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //4.如果该位置与待插入的key不相同，则判断当前是否为树节点
        else if (p instanceof TreeNode)
            //4.1 插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //当前节点不是树节点,遍历链表，同时计数得链表长度
            for (int binCount = 0; ; ++binCount) {
                //如果遍历到的节点为null，说明已经到达链表尾部了
                if ((e = p.next) == null) {
                    //链表尾部插入节点
                    p.next = newNode(hash, key, value, null);
                    //判断当前链表长度是否大于8,执行treeifybin函数转为红黑树,并停止遍历
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果当前节点得hash和key都等于待插入的元素，则记录当前节点，停止循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果e不等于null，表明链表遍历没有到达末尾，存在key的映射
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //修改操作次数
    ++modCount;
    //hashmap长度加1，如果长度大于阈值，进行扩容
    if (++size > threshold)
        resize();
    //???看不懂
    afterNodeInsertion(evict);
    return null;
}
```



## treeifyBin(Node<K,V>[], int)

```java
/**链表转红黑树
 * Replaces all linked nodes in bin at index for given hash unless
 * table is too small, in which case resizes instead.
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //1.如果table数组为null或长度小于64，则不转成红黑树，而是进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //2.如果当前值不等于null，
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {//循环遍历原链表，创建新的树节点的链表
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        //将已经转成树节点的链表赋值给数组元素，将新链表转成红黑树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```



##  resize()

```Java
    /** 扩容机制
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //扩容前判断原table是否为null，如果是原容量为0，否则为table长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //1.判断原来的容量是否大于0，即判断是否为初始化或因链表过长扩容
    //1.1 旧表已初始化，容量大于0
    if (oldCap > 0) {
        //1.1.1如果原容量大于最大容量，将阈值设置为整型最大值，返回原数组
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //1.1.2 如果原容量左移1位没有达到最大值，同时原容量大于默认初始容量16，则将新的阈值设置为原阈值左移一位
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //旧表未初始化，但阈值大于0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //旧表未初始化，但阈值等于0
    else {               // zero initial threshold signifies using defaults
        //将新容量设置为16(默认容量),阈值为12(负载因子*默认容量)
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //如果是上诉第二种情况(即newThr未赋值)，新阈值设为（负载因子*默认容量）
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //如果原来的table不等于null，则进行数据迁移
    if (oldTab != null) {
        //遍历数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //如果旧数组当前元素不为null，则进行数据迁移
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //如果该元素没有链表，则将当前元素迁移到新计算的下标处
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //如果该元素为树节点
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果该元素有链表，进行遍历
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //判断原来hash值和旧容量相与是否为0
                        //如果为0，则不需要迁移，还在当前下标，将不需要迁移的拼成一条链
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //如果为1，则需要迁移到当前位置+oldCap处的位置，将需要迁移的拼成一条链
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        //如果存在不需要迁移的链，则将链放置在新数组相同的位置
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        //如果存在需要迁移的链，将需要迁移的链迁移到原位置+oldCap处的位置
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

## removeNode()

```java
/**
 * Implements Map.remove and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to match if matchValue, else ignored
 * @param matchValue if true only remove if value is equal
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

# 二、理解

## 1.hashmap put的流程，

这个图挺好的，从牛客网上复制的，侵删

![img](https://static.nowcoder.com/images/activity/2021jxy/java/img/hashmap-4.png)

## 2.HashMap有什么特点

线程不安全  可以使用null作为key和value

## 3.HashMap为什么用红黑树而不用B树等其他树

HashMap原本是数组加链表，链表由于查询慢的特点，需要查找效率更高的树结构来代替。而如果用B树，在数据量不多的情况下，数据都会挤在一个节点，这个时候遍历效率就退化成了链表。而avl树的查找效率虽然高，但是在插入和删除方面速度较慢。

## 4.HashMap的扩容机制

1. 数组的初始容量是16，而容量是以2的次方扩充的，

* 为了能够用位运算代替取模运算
* 提高性能，使用足够大的数组

2. 数组扩容是根据负载因子决定的，如果当前元素个数大于阈值时，则进行扩容
3. 当链表长度大于8时，会将链表转成红黑树，当链表长度缩小到另一个阈值时（6），又会将红黑树转换回单向链表提高性能

## 5.HashMap为什么线程不安全

在1.7以前，hashmap扩容是以头插法的形式将旧链表赋值到新数组，当并发执行操作时，会导致循环链表，从而引起死循环

在1.8之后，hashmap扩容采用尾插法，解决了死循环的问题，但是还有数据覆盖的问题。并发执行put操作时，如果上一个线程判断完该下标位置为null，准备插入时，线程被挂起，下一个线程同样判断该位置为null并插入。当上一个线程再次获取时间片，因为之前已经判断过该位置为null，则会重新赋值给该位置，导致上一线程的数据被覆盖了。

## 6.HashMap、LinkedHashMap、Hashtable的区别

Hashtable是线程安全的，HashTable不允许使用null为键，但是允许value为null，Hashtable是一个古老的类，不建议使用，性能太低

HashMap是线程不安全的，允许使用一个null为键，可以有多个null值，

LInkedHashMap是HashMap的子类，底层维护entry数组，在HashMap的基础上给entry增加了before和after，相当于HashMap+双链表，所有LinkedHashMap可以根据插入顺序进行访问 