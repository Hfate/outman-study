# B+Tree

   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/stickPicture.png)
+ B+跟B树不同B+树的非叶子节点不保存关键字记录的指针，这样使得B+树每个节点所能保存的关键字大大增加；
+ B+树叶子节点保存了父节点的所有关键字和关键字记录的指针，每个叶子节点的关键字从小到大链接；
+ B+树的根节点关键字数量和其子节点个数相等;
+ B+的非叶子节点只进行数据索引，不会存实际的关键字记录的指针，所有数据地址必须要到叶子节点才能获取到，所以每次数据查询的次数都一样；

### 平衡二叉树并不适合作为索引结构
    树的深度比B树高，需要频繁io；逻辑结构上的平衡二叉树，其物理实现是数组。然后由于在逻辑结构上相近的节点在物理结构上可能会差很远；

###B树特点
     B树的每个节点可以存储多个关键字，它将节点大小设置为磁盘页的大小，充分利用了磁盘预读的功能；
     
### B+树比B树更适合做索引的原因
    B树：有序数组+平衡多叉树； 
    B+树：有序数组链表+平衡多叉树；

   + B+树的关键字全部存放在叶子节点中，非叶子节点用来做索引，而叶子节点中有一个指针指向一下个叶子节点。做这个优化的目的是为了提高区间访问的性能。而正是这个特性决定了B+树更适合用来存储外部数据。
   + 因为非叶子节点不存储数据，故可以存储更多的关键字，即映射更多的磁盘空间，树得深度比B树小；查找性能更快
   
###优化
    B树也好B+树也好，根或者上面几层因为被反复query，所以这几块基本都在内存中，不会出现读磁盘IO，一般已启动的时候，就会主动换入内存        