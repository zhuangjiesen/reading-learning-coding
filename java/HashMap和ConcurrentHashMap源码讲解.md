# HashMap和ConcurrentHashMap源码讲解

### HashMap 
#### put :
```
1.判断 table 是否空，空则初始化
2.计算槽位
3.槽位首节点如果为空直接插入槽
4.槽位首节点不为空
	4.1 判断槽位首节点的 key 相同 
		替换 value 
    4.2 槽位首节点的 key 相同 
        4.2.1 节点是红黑树节点 (返回旧节点)
              进红黑树查找
		4.2.2 节点是链表节点  (返回旧节点)
				4.2.2.1 判断节点的 next 是否空
					接到链表最尾
					长度到达7 触发转红黑树结构
					跳出循环
				4.2.2.1 判断节点 key 相同
					跳出循环(说明已经有值)

        4.2.3 判断旧节点是否为空(说明表中已经存在相同key )
        	    4.2.3.1 旧值的value 替换成新的 value 
        	    4.2.3.2 触发 afterNodeAccess() 方法
        	    3.2.3.3 返回旧值value 
        	    4.2.3.4 put方法结束
        4.2.4 判断旧节点为空
        	    4.2.4.1 修改记录modCount 加 1 
        	    4.2.4.2 判断size 加 1 并判断大小是不是超过临界值，进行扩容
        	    4.2.4.3 触发afterNodeInsertion() 方法
        	    4.2.3.4 返回 null 
        	    4.2.4.5 put方法结束
```

### ConcurrentHashMap 1.8源码




### 初始化 (处理并发)

```
在 putVal() 方法中，首先套了个(死)循环，每次都获取最新的 table 引用，然后在第一次对数据进行put 时，判断table 和 sizeCtl (数组大小 ：-1 表示初始化 -N 表示进行扩容的线程数为N-1 ， 初始化完成后数值是容量的0.75倍也就是thredholds 值)
initTable() 中：


    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        // 死循环完成初始化
        while ((tab = table) == null || tab.length == 0) {
        	// 判断siezCtl(-1) 只允许一个线程进行初始化，其他的交出执行权
            if ((sc = sizeCtl) < 0)
                //交出执行权，这里可能会产生额外的上下文切换消耗
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                // CAS设置 sizeCtl 为 -1 表示初始化状态
            	// 初始化数组 table 
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // 位运算 把 sizeCtl 设置成table 大小的0.75
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

### 1.容量计算 （处理并发）：
#### addCount() 方法
```
int 变量 baseCount ，通过cas 去更新容量,如果遇到并发cas 返回false ，则就继续通过CAS 更新 CounterCell[] counterCells 的value  ，用来继续更新长度，降低自旋锁的自旋消耗(毕竟对一个变量的CAS循环，比对一堆变量的CAS概率小)
```
#### sumCount() 方法统计长度
```
遍历 counterCells 把value值相加,再加上baseCount  ，计算得long 值转成int 型输出

```

#### 源码：

```

    /**
     * Adds to count, and if table is too small and not already
     * resizing, initiates transfer. If already resizing, helps
     * perform transfer if work is available.  Rechecks occupancy
     * after a transfer to see if another resize is already needed
     * because resizings are lagging additions.
     *
     * @param x the count to add
     * @param check if <0, don't check resize, if <= 1 only check if uncontended
     */
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        /*
 			一般 counterCells 是空的，所以总是执行 后面的cas 操作，如果是true 就直接返回了
 			如果是并发插入，其中一个线程cas 操作失败返回false ,则触发counterCells 的操作。
 			这里的设计理念我的理解是分两级(毕竟并发的风险不高)的进行计算，保证效率，防止自旋 , 所以进入 fullAddCount() 里面是自旋的
        */
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                /*
                    CAS 失败后转入更新 counterCells ，防止CAS 自旋的问题
                */
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        /*
            用来判断是否扩容
        */
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                /*
                    当 sizeCtl 小于 0 进入扩容状态后，其他线程只需要拿到 nextTable 进入transfer() 方法 ，就可以触发并发扩容
                */ 
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        /*
                            已经处于扩容状态下，直接传入 nextTable 进行触发 并发扩容 操作
                        */
                        transfer(tab, nt);
                }
                // 将 sizeCtl 设置成 -2 说明进入扩容状态
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                // sumCount() 把 baseCount 和 countCells 的元素加起来，转成int 型
                s = sumCount();
            }
        }
    }

```


### put() 

```
跟hashmap一样
1.判断是否初始化，初始化
2.判断槽位首节点是否空，空则直接插入槽位(CAS),失败了说明首节点已经被其他线程插入，返回循环，继续判断
3.首节点不为空，判断首节点的hash值是不是 -1 (MOVED) ，用来判断是不是正在扩容，正在扩容触发helpTransfer();
4.首节点不为空，synchronized 锁住首节点(之后还会进行个double check 判断首节点的流程，保证线程安全),按照hashMap 那样插入表(红黑树之类)中， 返回oldValue 
5.如果oldValue 不为空直接返回
6.如果oldValue 空，则触发addCount() ，计算长度 

```

#### 源代码：
```


    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p>The value can be retrieved by calling the {@code get} method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with {@code key}, or
     *         {@code null} if there was no mapping for {@code key}
     * @throws NullPointerException if the specified key or value is null
     */
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;

       	// 死循环
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //判断数组是否初始化
            if (tab == null || (n = tab.length) == 0)
            	// 并发初始化
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            	// 找到槽位首节点 
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    // CAS(无锁操作) 插入槽位，成功即返回，失败则继续循环
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
            	// 判断 forwardingNode (hash 为 -1 key value 都是null 的用来作并发扩容的标记节点，也许ConcurrentHashMap 的 key value 不为空的原因是这样插入数据没有意义，另外是不是因为已经用了forwardingNode 来作标记节点)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                /*
                    事实上保证线程安全的最重要的点是这里，锁住了槽位，通过double check 方式保证线程阻塞时的状态
                    之后就是类似HashMap 的常规操作了
                */
                synchronized (f) {
                	// double check 锁住首节点
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            // 同hashMap 的插入
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
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
                // 如果是覆盖操作，直接返回，所以不会执行到addCount();
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 记录baseCount 与 countCells 并判断扩容
        addCount(1L, binCount);
        return null;
    }


```

### 扩容(并发)

transfer() 方法

```
关键点 
1. sizeCtl
2. nextTable
3. transferIndex
4. forwardingNode 

讲解：
1. 这是在addCount() 中的 ，首先判断完容量到达临界时，设置sizeCtl 为 -2 (扩容状态)
2. 其他线程自旋等待 sizeCtl 和 nextTable 的值，当符合时，进入 transfer
3. transfer() 方法的原理就是 通过 transferIndex (从大到小) 记录正在扩容的槽位，这个变量当然是用CAS 控制原子操作，所以每个线程都会拿到一个自己的槽位进行扩容，完成了并发扩容的控制

transferIndex 用来记录(标记)当前正在扩容的槽位， 初始是从最大值 table.length 到 0 遍历的
遍历数组
1.首节点为空直接设置 forwardingNode 
2.首节点就是 forwardingNode ，继续循环
3.需要进行处理的节点，synchronized 锁住首节点(紧接着来个double check)
也是跟普通的HashMap 扩容差不多，分成两条链(两个槽位，通过hash值高位多个1进行判断)
最后不同是用cas 设置到各自的槽位，然后将原来的 table 的这个槽位设置成forwardingNode , 让别的线程并发扩容 (别的线程 到这里获取到的首节点可能就不一样了，所以就会重新循环)




```



源代码：

```

/**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                // 初始化新table ，原来的两倍
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
        /*
            初始化扩容节点
            这个fwd 刚new 出来 还没有设置到旧槽位，说明数据还是在旧的数组上，所以get 的时候直接可以从table 中拿，虽然都是继承 Node ，实现find() 方法，最后只需要各自实现find() 方法，就能保证无锁的读
            
        */ 
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        /*
            用来判断是否扩容完成

        */
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
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
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    /*
                        也是一个double check 保证了只有一个线程扩容这个槽。
                        底下的操作跟HashMap 1.8的扩容几乎一样了。就是分成2个链表，一个还在原来的位置，一个扩容至新槽位(当前位置＋旧容量)，然后就通过CAS插入 nextTable 并把 forwardingNode 设置到这个槽位上
                        这个点也很严谨，先设置了nextTable ，然后设置 fwd 保证了 get() 方法查询的时候的严谨和无锁

                        接着就是红黑树啥的。不说了介绍HashMap 的帖子都会说红黑树
                    */
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






### containsValue() 查看value存在


```
    /**
     * Returns {@code true} if this map maps one or more keys to the
     * specified value. Note: This method may require a full traversal
     * of the map, and is much slower than method {@code containsKey}.
     *
     * @param value value whose presence in this map is to be tested
     * @return {@code true} if this map maps one or more keys to the
     *         specified value
     * @throws NullPointerException if the specified value is null
     */
    public boolean containsValue(Object value) {
        if (value == null)
            throw new NullPointerException();
        Node<K,V>[] t;
        if ((t = table) != null) {
            // 封装了一个类似迭代器的东西 it.advance() 将value封装在一起，就是访问下一个节点
            Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
            for (Node<K,V> p; (p = it.advance()) != null; ) {
                V v;
                if ((v = p.val) == value || (v != null && value.equals(v)))
                    return true;
            }
        }
        return false;
    }


        /**
            遍历map 的table 获取节点
         * Same as Traverser version
         */
        final Node<K,V> advance() {
            Node<K,V> e;
            if ((e = next) != null)
                e = e.next;
            for (;;) {
                Node<K,V>[] t; int i, n;
                if (e != null)
                    return next = e;
                if (baseIndex >= baseLimit || (t = tab) == null ||
                    (n = t.length) <= (i = index) || i < 0)
                    return next = null;
                /*
                    正在扩容中，可以直接替换tab ,这边有个很妙的点，就是这个循环是从最小开始的，
                    扩容是从最大开始的，所以这里如果遇到 forwadingNode 说明之后的节点肯定是扩容成功的，所以可以直接替换tab 
                */ 
                if ((e = tabAt(t, i)) != null && e.hash < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        e = null;
                        pushState(t, i, n);
                        continue;
                    }
                    else if (e instanceof TreeBin)
                        e = ((TreeBin<K,V>)e).first;
                    else
                        e = null;
                }
                if (stack != null)
                    recoverState(n);
                else if ((index = i + baseSize) >= n)
                    index = ++baseIndex;
            }
        }



```

#### 提一句，containsKey() 方法：

```
    /**
     * Tests if the specified object is a key in this table.
     *
     * @param  key possible key
     * @return {@code true} if and only if the specified object
     *         is a key in this table, as determined by the
     *         {@code equals} method; {@code false} otherwise
     * @throws NullPointerException if the specified key is null
     */
    public boolean containsKey(Object key) {
        return get(key) != null;
    }

```





# ConcurrentHashMap 的读为什么不用加锁
```
首先，ConcurrentHashMap 的数据都是存在变量 table 或者 nextTable ,要嘛数据在 table 要嘛在 nextTable 中，所以用了关键字 volatile 保证了访问变量的内存可见性，然后在 get() 时，判断hash 是不是等于 -1 (forwardingNode) ，因为创建 forwardingNode 时，已经把nextTable 放进它的变量，所以查到hash 等于 -1 时，会直接进入nextTable 搜索。所以get() 时不用加锁




源码如下：


    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                /*
                    
                */
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }




```





### 最后想说

```

ConcurrentHashMap 的 put 并不是因为CAS保证了线程安全，而是因为用了double check 加上槽位头结点上锁，完成了线程安全的操作，而CAS只是常规操作，当然也是线程安全的实现，但是我个人认为真正关键的位置就是那个 putVal() 方法下的 synchronized (f) ... 代码块

```




