# 一、源码



## jdk 1.8

> put方法流程和hashmap一样，
>
> 区别：
>
> * 是因为ConcurrentHashMap的扩容机制是可以并发扩容的，在判断节点是需要额外判断节点的hash值，在遍历链表时需要加锁
> * hashmap允许null键null值，ConcurrentHashMap不允许null键null值
>
> 并发扩容

### 成员变量

```java
transient volatile Node<K,V>[] table;
private static final int DEFAULT_CAPACITY = 16;
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
//以上变量和HashMap相同

private transient volatile int sizeCtl;//表初始化和扩容控制
private transient volatile Node<K,V>[] nextTable;
//扩容时的新表，并发扩容时所有线程都协助将数据迁移到这个表内

```

### putVal(K, V, boolean)

> 不允许插入null键null值
>
> 插入过程中如果检测到了forward节点(有该节点说明其他线程正在扩容)会协助其他线程扩容
>
> 遍历链表节点时会上锁

```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //如果key、value为null，抛出空指针异常
    if (key == null || value == null) throw new NullPointerException();
    //hashcode扰动，使hash值分布得更平均
    int hash = spread(key.hashCode());
    int binCount = 0;
    //遍历hashmap，死循环，通过条件中断循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //未初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //根据hash值获取下标，如果这个位置没有值，则使用CAS插入节点，不需要加锁
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果当前节点的hash值为MOVED，即有其他线程正在扩容，则协助扩容，处理完之后再进入for循环进行判断
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //给当前Node节点上锁
            synchronized (f) {
                //只有节点未更改
                if (tabAt(tab, i) == f) {
                    //fh：当前Node节点的hash值.如果大于0，表示
                    if (fh >= 0) {
                        binCount = 1;
                        //遍历链表所有节点
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果遍历到的节点，key值和hash值相等，记录旧val值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //如果map允许覆盖，则修改val值
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //如果遍历到最后一个节点，则新节点就插入到末尾，结束遍历
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //fh<0,则判断当前节点是否为TreeBin，相当于hashmap中的treeNode，
                    //按照树的方式，插入节点
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
            //binCount!=0说明进行了链表遍历
            if (binCount != 0) {
                //如果链表长度大于8，进行树化
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //如果旧值不为null，说明修改了，返回旧值
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //将当前concurrentHashMap的数量加1
    addCount(1L, binCount);
    return null;
}
```



### initTable

> 初始化table

```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    //循环判断table是否为空或长度为0
    while ((tab = table) == null || tab.length == 0) {
        //如果sizeCtl小于0，表明有其他线程正在初始化，将线程挂起
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //使用CAS将sizeCtrl设为-1，表明当前线程已经开始初始化了
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                //重新判断table是否为空或长度为0，
                if ((tab = table) == null || tab.length == 0) {
                    //初始容量16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



### addCount()

> 该方法用于将ConcurrentHashMap的数量加1，
>
> 如果表太小且没有进行扩容，则启动扩容
>
> 如果已经在扩容，则协助执行扩容
>
> 扩容后重新检查数组大小，以查看其他扩容已经需要开始

```java
/**
 * Adds to count, and if table is too small and not already
 * resizing, initiates transfer. If already resizing, helps
 * perform transfer if work is available.  Rechecks occupancy
 * after a transfer to see if another resize is already needed
 * because resizings are lagging additions.
 *
 * @param x the count to add 添加的数量
 * @param check if <0, don't check resize, if <= 1 only check if uncontended
 * 如果check小于0，不检查扩容，如果小于1仅检查是否无竞争
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //利用CAS方法更新baseCount的值
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
        s = sumCount();
    }
    //如果check大于等于0，则需要检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //判断
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
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
            s = sumCount();
        }
    }
}
```





### transfer(Node<K,V>[] , Node<K,V>[])

`Node<K,V>[] tab`  需要扩容的旧表

`Node<K,V>[] nextTab` 数据迁移的新表(如果是第一个进行扩容的则为null)

> **数据迁移(扩容) ** 相当于HashMap中的resize()
>
> 1. 如果nextTable为空，构建一个容量为旧表2倍的新表
> 2. 如果finishing标识符为true，表明已经完成了所有节点的复制，将新表赋给table
> 3. 遍历旧表
>    * 如果节点为null，则放入一个forward节点，表示已处理过
>    * 如果节点为forward，则跳过该节点
>    * 否则，节点上锁，将数据迁移到新表

```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //构建一个nextTab，
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //新表是旧表的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    //标识符，确保提交nextTab前进行扫描
    boolean finishing = false; // to ensure sweep before committing nextTab
    //遍历
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //这个while循环体的作用就是在控制i--  通过i--可以依次遍历原hash表中的节点
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果finishing为true，表明所有节点都完成了复制，将nextTable赋值为table，
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //CAS更新扩容阈值
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //如果当前节点为null，则放入forward节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //如果当前节点为forward节点，表示已经处理过了，跳过这个节点
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //节点上锁，开始将旧表数据迁移到新表
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
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
                    //如果当前节点是树
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



### helpTransfer(Node<K,V>[], Node<K,V>)

`Node<K,V>[] tab` :需要数据转移的表，用于再次判断当前表是否为null

`Node<K,V> f`:开始数据转移的节点，用于再次判断是否为forward节点

> 协助数据转移
>
> 当线程进行插入操作，如果发现当前节点为forward，则表明有其他线程正在数据迁移(扩容),应当协助数据转移

```java
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //执行前再次判断，如果当前当前表不为null、当前节点为forward节点、数据迁移的新表不为null，则开始协助数据迁移
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
       
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //如果
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //CAS设置sizeCtrl加1,表明新增一个线程参与数据迁移(扩容)，开始协助扩容
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



## jdk 1.7

> 在1.7中，ConcurrentHashMap底层是以Segment数组+HashEntry数组+链表的形式，其中Segment数组继承了ReentrantLock，负责控制并发线程，将HashEntry分段，每一段存储在Segment中

### 关键成员变量

```java
final Segment<K,V>[] segments;//
final int segmentShift;//segment段内索引偏移值
final int segmentMask;//
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;//最小segment数量
static final int MAX_SEGMENTS = 1 << 16;//最大segment数量

//同hashmap：初始容量、最大容量、负载因子
static final int DEFAULT_INITIAL_CAPACITY = 16;
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//默认并发度:最多可同时操作线程数
static final int DEFAULT_CONCURRENCY_LEVEL = 16;



```



### 构造函数

> 1.7的ConcurrentHashMap主要是3参构造函数,其他2个重载的构造函数也是调用这个3参的构造函数

`initialCapacity`:初始容量

`loadFactor`:负载因子

`concurrencyLevel`:并发度

```Java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    //三个参数要非负
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    //并发度最多只能到MAX_SEGMENTS,因为并发是由Segment类控制的
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    //如果初始化segment长度为刚好大于并发度的2次幂：类似于HashMap的容量为大于入参的2次幂
    //ssize：segment数组长度
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    //容量最大只能为MAXIMUM_CAPACITY
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    //将初始容量的HashTable数组进行分段，指定每个Segment元素应该包含的HashEntry数量，即键值对的数量
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    //如果segment中的HashEntry数量 > 最小数量MIN_SEGMENT_TABLE_CAPACITY
    //将实际segment包含的hashEntry数量提高到刚好大于原来的2次幂
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    //创建segment数组和segment[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    //写入segment[0],即初始化第一个segment:
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```



###  ConcurrentHashMap：  put(K key, V value)

```java
/**
 * Maps the specified key to the specified value in this table.
 * Neither the key nor the value can be null.
 *
 * <p> The value can be retrieved by calling the <tt>get</tt> method
 * with a key that is equal to the original key.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>
 * @throws NullPointerException if the specified key or value is null
 */
@SuppressWarnings("unchecked")
public V put(K key, V value) {
    Segment<K,V> s;
    //null值不允许，抛出空指针异常
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    //根据hash获取索引
    int j = (hash >>> segmentShift) & segmentMask;
    //找到索引对应的segment
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    //调用segment的put方法
    return s.put(key, hash, value, false);
}
```



### Segment：                   put(K key, int hash, V value, boolean onlyIfAbsent)

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //此时是在单个segment操作元素，而1.7的ConcurrentHashMap就是以分段锁控制并发，开头需要对segment加锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;//根据hash获取下标
        HashEntry<K,V> first = entryAt(tab, index);//根据下标获取HashEntry节点
        
        //遍历当前HashEntry链表
        for (HashEntry<K,V> e = first;;) {
            //如果当前HashEnrey存在元素
            if (e != null) {
                K k;
                //如果key值value值相等，记录当前旧值
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    //如果onlyIfAbsent为false，则覆盖旧值
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            //如果当前遍历到null
            else {
                //如果链头不为null，说明此时已经遍历到了链尾，
                if (node != null)
                    node.setNext(first);
                //否则，新建HashEntry作为链头
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                //segment中的HashEntry数量加1
                int c = count + 1;
                //如果HashEntry数量大于阈值，table长度大于最大容量，则进行rehash扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



### Segment : rehash(HashEntry<K,V> node)

```java
/**
 * Doubles size of table and repacks entries, also adding the
 * given node to new table
 */
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    /*
     * Reclassify nodes in each list to new table.  Because we
     * are using power-of-two expansion, the elements from
     * each bin must either stay at same index, or move with a
     * power of two offset. We eliminate unnecessary node
     * creation by catching cases where old nodes can be
     * reused because their next fields won't change.
     * Statistically, at the default threshold, only about
     * one-sixth of them need cloning when a table
     * doubles. The nodes they replace will be garbage
     * collectable as soon as they are no longer referenced by
     * any reader thread that may be in the midst of
     * concurrently traversing table. Entry accesses use plain
     * array indexing because they are followed by volatile
     * table write.
     */
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;//扩容2倍
    threshold = (int)(newCapacity * loadFactor);//重新计算阈值
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    
    //遍历旧表
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;//计算在新表的下标
            //如果只有链头，则链头转移到新表相应位置
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            //否则，需要对链表遍历
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                //遍历链表重新hash计算每一个节点的新下标
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;//计算节点在新表的下标
                    //如果新旧下标不相同，则使用新下标
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```



# 二、面试题

## 1.ConcurrentHashMap 的底层实现原理

在Java8中  ConcurrentHashMap底层是以数组+链表+红黑树的形式，通过CAS和synchronized来实现线程安全，

在Java7中 ConcurrentHashMap底层是以Segment数组+HashEntry数组+HashEntry链表的形式，使用分段锁实现线程安全

## 2.Java7和8中ConcurrentHashMap的区别

1. `结构`：在7中底层是通过Segment数组+HashEntry数组+HashEntry链表的结构，Segment继承了ReentrantLock，通过分段锁实现了线程安全，而在8中则是使用了和8中hashmap相同的结构，也就是数组+链表+红黑树的形式，移除了segment，使锁的粒度更小了，通过CAS和synchronized来实现线程安全。

2. `扩容`7中的扩容是在segment中进行的，因为7中是分段锁的，各负责一段区域，而8中是可以并发扩容，在检测到有线程扩容会停止插入并协助扩容，大大提高了效率。

3. `get`:get操作没有什么改变，都是对value添加了valotile，不需要加锁。

4. `size()获取容量`：在7中获取容量是无锁遍历两次，如果结果相同就返回结果，否则上锁遍历。而在8中是用baseCount存储当前节点个数
5. `put`在7中put是需要计算两次hash，先定位segment，加锁，然后再定位桶，如果有其他线程竞争，最多自旋64次就会挂起。在8中put操作会判断当前节点是否为forward节点，如果是则协助其他线程扩容，否则就对锁住当前节点，如果节点没变就遍历链表，否则判断是不是树节点，再遍历插入。

## 3.为什么ConcurrentHashMap不能有null键null值

没办法分辨get得到的null是null值还是因为键不存在返回的null，因为这在多线程是模糊的，而单线程可以有，是因为可以使用containsKey()查询，但在多线程没办法保证containsKey()前后查询的结果是一致的，

## 4.为什么Java 8中使用CAS+synchronized，不是使用CAS+reentrantLock

synchronized可以很方便地锁住链表头节点，同时在1.6时synchronized做了优化，性能上和reentrantLock差不多，但是使用reentrantLock需要Node继承reentrantLock，但是需要同步地仅仅是头节点，这样增加了内存开销