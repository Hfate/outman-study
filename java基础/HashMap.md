# HashMap

    主干部分是一个entry数组，采用数组+链表的结构。
   
   ![HashMap](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/20181102221702492.png)
   
   + 重要字段
   ```
        /**实际存储的key-value键值对的个数*/
        transient int size;
        
        /**阈值，当table == {}时，该值为初始容量（初始容量默认为16）；当table被填充了，也就是为table分配内存空间后，
        threshold一般为 capacity*loadFactory。HashMap在进行扩容时需要参考threshold*/
        int threshold;
        
        /**负载因子，代表了table的填充度有多少，默认是0.75
        加载因子存在的原因，还是因为减缓哈希冲突，如果初始桶为16，等到满16个元素才扩容，某些桶里可能就有不止一个元素了。
        所以加载因子默认为0.75，也就是说大小为16的HashMap，到了第13个元素，就会扩容成32。
        */
        final float loadFactor;
        
        /**HashMap被改变的次数，由于HashMap非线程安全，在对HashMap进行迭代时，
        如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），
        需要抛出异常ConcurrentModificationException*/
        transient int modCount;
   ``` 


   + put操作
    
     ```
     //实现 put 和相关方法
     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
         Node<K,V>[] tab; Node<K,V> p; int n, i;
     
         //如果table为空或者长度为0，则进行resize()（扩容）
         if ((tab = table) == null || (n = tab.length) == 0)
             n = (tab = resize()).length;
     
         //确定插入table的位置，算法是上面提到的 (n - 1) & hash，在 n 为 2的次幂的时候，相当于取模操作
         if ((p = tab[i = (n - 1) & hash]) == null)
             //找到key值对应的位置并且是第一个，直接插入
             tab[i] = newNode(hash, key, value, null);
     
         //在table的 i 的位置发生碰撞，分两种情况
         //1、key值是一样的，替换value值
         //2、key值不一样的
         //而key值不一样的有两种处理方式：1、存储在 i 的位置的链表 2、存储在红黑树中
         else {
             Node<K,V> e; K k;
     
             //第一个Node的hash值即为要加入元素的hash
             if (p.hash == hash &&
                 ((k = p.key) == key || (key != null && key.equals(k))))
                 e = p;
             else if (p instanceof TreeNode)
                 e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
             else {
                 //如果不是TreeNode的话，即为链表，然后遍历链表
                 for (int binCount = 0; ; ++binCount) {
     
                     //链表的尾端也没有找到key值相同的节点，则生成一个新的Node
                     //并且判断链表的节点个数是不是到达转换成红黑树的上界达到，则转换成红黑树
                     if ((e = p.next) == null) {
     
                         //创建链表节点并插入尾部
                         p.next = newNode(hash, key, value, null);
     
                         //超过了链表的设置长度（默认为8）则转换为红黑树
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
     
             //如果e不为空
             if (e != null) { // existing mapping for key
                 //就替换就的值
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
     
   + 扩容
       
      ```
        final Node<K,V>[] resize() {
            Node<K,V>[] oldTab = table;
            int oldCap = (oldTab == null) ? 0 : oldTab.length;
            int oldThr = threshold;
            int newCap, newThr = 0;
        
            //判断Node的长度，如果不为零
            if (oldCap > 0) {
                //判断当前Node的长度，如果当前长度超过 MAXIMUM_CAPACITY（最大容量值）
                if (oldCap >= MAXIMUM_CAPACITY) {
                    //新增阀值为 Integer.MAX_VALUE
                    threshold = Integer.MAX_VALUE;
                    return oldTab;
                }
        
                //如果小于这个 MAXIMUM_CAPACITY（最大容量值），并且大于 DEFAULT_INITIAL_CAPACITY （默认16）
                else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                    //进行2倍扩容
                    newThr = oldThr << 1; // double threshold
            }
            else if (oldThr > 0) // initial capacity was placed in threshold
                //指定新增阀值
                newCap = oldThr;
        
            //如果数组为空
            else {               // zero initial threshold signifies using defaults
                //使用默认的加载因子(0.75)
                newCap = DEFAULT_INITIAL_CAPACITY;
                //新增的阀值也就为 16 * 0.75 = 12
                newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
            }
            if (newThr == 0) {
                //按照给定的初始大小计算扩容后的新增阀值
                float ft = (float)newCap * loadFactor;
                newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                          (int)ft : Integer.MAX_VALUE);
            }
        
            //扩容后的新增阀值
            threshold = newThr;
            @SuppressWarnings({"rawtypes","unchecked"})
            //扩容后的Node数组
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
            table = newTab;
        
            //如果数组不为空，将原数组中的元素放入扩容后的数组中
            if (oldTab != null) {
                for (int j = 0; j < oldCap; ++j) {
                    Node<K,V> e;
                    if ((e = oldTab[j]) != null) {
                        oldTab[j] = null;
        
                        //如果节点为空，则直接计算在新数组中的位置，放入即可
                        if (e.next == null)
                            newTab[e.hash & (newCap - 1)] = e;
                        else if (e instanceof TreeNode)
                            //拆分树节点
                            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                        else { // preserve order
                            //如果节点不为空，且为单链表，则将原数组中单链表元素进行拆分
                            Node<K,V> loHead = null, loTail = null;//保存在原有索引的链表
                            Node<K,V> hiHead = null, hiTail = null;//保存在新索引的链表
                            Node<K,V> next;
                            do {
                                next = e.next;
        
                                //哈希值和原数组长度进行&操作，为0则在原数组的索引位置
                                //非0则在原数组索引位置+原数组长度的新位置
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
      ```      