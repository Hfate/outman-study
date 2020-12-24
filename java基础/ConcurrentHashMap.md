# ConcurrentHashMap
    
   + 数据结构
   ![ConcurrentHashMap](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/imD4hUB.png) 
   
   + 添加元素 put
   
     ```
        final V putVal(K key, V value, boolean onlyIfAbsent) {
                if (key == null || value == null) throw new NullPointerException();
                int hash = spread(key.hashCode());   //@1，讲解见下面小标题。
                 //i处结点数量，2: TreeBin或链表结点数, 其它：链表结点数。主要用于每次加入结点后查看是否要由链表转为红黑树
                int binCount = 0; 
                for (Node<K,V>[] tab = table;;) {   //CAS经典写法，不成功无限重试，让再次进行循环进行相应操作。
                    Node<K,V> f; int n, i, fh;
                    //除非构造时指定初始化集合，否则默认构造不初始化table，所以需要在添加时元素检查是否需要初始化。
                    if (tab == null || (n = tab.length) == 0)
                        tab = initTable();  //@2
                    //CAS操作得到对应table中元素
                    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //@3
                        if (casTabAt(tab, i, null,
                                     new Node<K,V>(hash, key, value, null)))
                            break;                   //null创建Node对象做为链表首结点
                    }
                    else if ((fh = f.hash) == MOVED)  //当前结点正在扩容
                    	//让当前线程调用helpTransfer也参与到扩容过程中来，扩容完毕后tab指向新table。
                        tab = helpTransfer(tab, f); 
                    else {
                        V oldVal = null;
                        synchronized (f) {
                            if (tabAt(tab, i) == f) {  //双重检查i处结点未变化
                                if (fh >= 0) {  //表明是链表结点类型，hash值是大于0的，即spread()方法计算而来
                                    binCount = 1;
                                    for (Node<K,V> e = f;; ++binCount) {
                                        K ek;
                                        if (e.hash == hash &&
                                            ((ek = e.key) == key ||
                                             (ek != null && key.equals(ek)))) {
                                            oldVal = e.val;
                                            //onlyIfAbsent表示是新元素才加入，旧值不替换，默认为fase。
                                            if (!onlyIfAbsent)  
                                                e.val = value;
                                            break;
                                        }
                                        Node<K,V> pred = e;
                                        if ((e = e.next) == null) {
        	                                //jdk1.8版本是把新结点加入链表尾部，next由volatile修饰
                                            pred.next = new Node<K,V>(hash, key,
                                                                      value, null);
                                            break;
                                        }
                                    }
                                }
                                else if (f instanceof TreeBin) {  //红黑树结点类型
                                    Node<K,V> p;
                                    binCount = 2;
                                    if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                                   value)) != null) {   //@4
                                        oldVal = p.val;
                                        if (!onlyIfAbsent)
                                            p.val = value;
                                    }
                                }
                            }
                        }
                        if (binCount != 0) {
                            if (binCount >= TREEIFY_THRESHOLD)  //默认桶中结点数超过8个数据结构会转为红黑树
                                treeifyBin(tab, i);   //@5
                            if (oldVal != null)
                                return oldVal;
                            break;
                        }
                    }
                }
                addCount(1L, binCount);  //更新size，检测扩容
                return null;
            }
     ``` 
     
   + 初始化哈希表
     
     ``` 
        private final Node<K,V>[] initTable() {
                Node<K,V>[] tab; int sc;
                while ((tab = table) == null || tab.length == 0) {
                    if ((sc = sizeCtl) < 0)
                        Thread.yield(); 
                        //正在初始化时将sizeCtl设为-1
                    else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                        try {
                            if ((tab = table) == null || tab.length == 0) {
                                int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  //DEFAULT_CAPACITY为16
                                @SuppressWarnings("unchecked")
                                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                                table = tab = nt;
                                sc = n - (n >>> 2);   //扩容阈值为新容量的0.75倍
                            }
                        } finally {
                            sizeCtl = sc;   //扩容保护
                        }
                        break;
                    }
                }
                return tab;
            }
     ``` 
   
   + 扩容
    
        + 使用put()添加元素时会调用addCount()，内部检查sizeCtl看是否需要扩容。
        + tryPresize()被调用，此方法被调用有两个调用点：
            + 链表转红黑树(put()时检查)时如果table容量小于64(MIN_TREEIFY_CAPACITY)，则会触发扩容。
            + 调用putAll()之类一次性加入大量元素，会触发扩容。  
            
        + transfer
                
          ```
            private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
                    int n = tab.length, stride;
                    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  //每个线程处理桶的最小数目，可以看出核数越高步长越小，最小16个。
                        stride = MIN_TRANSFER_STRIDE; // subdivide range
                    if (nextTab == null) {
                        try {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];  //扩容到2倍
                            nextTab = nt;
                        } catch (Throwable ex) {      // try to cope with OOME
                            sizeCtl = Integer.MAX_VALUE;  //扩容保护
                            return;
                        }
                        nextTable = nextTab;
                        transferIndex = n;  //扩容总进度，>=transferIndex的桶都已分配出去。
                    }
                    int nextn = nextTab.length;
                      //扩容时的特殊节点，标明此节点正在进行迁移，扩容期间的元素查找要调用其find()方法在nextTable中查找元素。
                    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab); 
                    //当前线程是否需要继续寻找下一个可处理的节点
                    boolean advance = true;
                    boolean finishing = false; //所有桶是否都已迁移完成。
                    for (int i = 0, bound = 0;;) {
                        Node<K,V> f; int fh;
                        //此循环的作用是确定当前线程要迁移的桶的范围或通过更新i的值确定当前范围内下一个要处理的节点。
                        while (advance) {
                            int nextIndex, nextBound;
                            if (--i >= bound || finishing)  //每次循环都检查结束条件
                                advance = false;
                            //迁移总进度<=0，表示所有桶都已迁移完成。
                            else if ((nextIndex = transferIndex) <= 0) {  
                                i = -1;
                                advance = false;
                            }
                            else if (U.compareAndSwapInt
                                     (this, TRANSFERINDEX, nextIndex,
                                      nextBound = (nextIndex > stride ?
                                                   nextIndex - stride : 0))) {  //transferIndex减去已分配出去的桶。
                                //确定当前线程每次分配的待迁移桶的范围为[bound, nextIndex)
                                bound = nextBound;
                                i = nextIndex - 1;
                                advance = false;
                            }
                        }
                        //当前线程自己的活已经做完或所有线程的活都已做完，第二与第三个条件应该是下面让"i = n"后，再次进入循环时要做的边界检查。
                        if (i < 0 || i >= n || i + n >= nextn) {
                            int sc;
                            if (finishing) {  //所有线程已干完活，最后才走这里。
                                nextTable = null;
                                table = nextTab;  //替换新table
                                sizeCtl = (n << 1) - (n >>> 1); //调sizeCtl为新容量0.75倍。
                                return;
                            }
                            //当前线程已结束扩容，sizeCtl-1表示参与扩容线程数-1。
                            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
            	                //还记得addCount()处给sizeCtl赋的初值吗？相等时说明没有线程在参与扩容了，置finishing=advance=true，为保险让i=n再检查一次。
                                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)   
                                    return;
                                finishing = advance = true;
                                i = n; // recheck before commit
                            }
                        }
                        else if ((f = tabAt(tab, i)) == null)
                            advance = casTabAt(tab, i, null, fwd);  //如果i处是ForwardingNode表示第i个桶已经有线程在负责迁移了。
                        else if ((fh = f.hash) == MOVED)
                            advance = true; // already processed
                        else {
                            synchronized (f) {  //桶内元素迁移需要加锁。
                                if (tabAt(tab, i) == f) {
                                    Node<K,V> ln, hn;
                                    if (fh >= 0) {  //>=0表示是链表结点
                                        //由于n是2的幂次方（所有二进制位中只有一个1)，如n=16(0001 0000)，第4位为1，那么hash&n后的值第4位只能为0或1。所以可以根据hash&n的结果将所有结点分为两部分。
                                        int runBit = fh & n;
                                        Node<K,V> lastRun = f;
                                        //找出最后一段完整的fh&n不变的链表，这样最后这一段链表就不用重新创建新结点了。
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
                                        //lastRun之前的结点因为fh&n不确定，所以全部需要重新迁移。
                                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                            int ph = p.hash; K pk = p.key; V pv = p.val;
                                            if ((ph & n) == 0)
                                                ln = new Node<K,V>(ph, pk, pv, ln);
                                            else
                                                hn = new Node<K,V>(ph, pk, pv, hn);
                                        }
                                        //低位链表放在i处
                                        setTabAt(nextTab, i, ln);
                                        //高位链表放在i+n处
                                        setTabAt(nextTab, i + n, hn);
                                        setTabAt(tab, i, fwd);  //在原table中设置ForwardingNode节点以提示该桶扩容完成。
                                        advance = true;
                                    }
                                    else if (f instanceof TreeBin) { //红黑树处理。
                                        ...
          ```