# 概述
    
    无论是复杂的现实世界还是复杂的计算机系统，性能调优始终是一门艺术，它建立在知识与经验之上；
    java性能调优主要探讨两个点：
    1 ，如何通过JVM的配置来影响程序的各项性能指标；
    2 ，理解java平台特性（java语音以及java API）对性能的影响；

## 性能调优工具
    
   + 操作系统工具
       + CPU使用率
         ```
         vmstat 1
         ```
         性能调优的目的是尽可能短的时间内让CPU使用率更高
       + CPU运行队列（可运行队列长度）
         ```
         c:> typeperf -si 1 "\System\Processor Queue Length"
         ```
         如果试图运行的线程数超过可用CPU，系统过载，性能就会下降
       + 磁盘使用率
         ```
         iostat -xm 5
         ```
         对于所有应用来说，即便不直接写磁盘，系统交换仍会影响它们的性能
         写入磁盘的应用遇到瓶颈，要么写入效率不高（吞吐太低），要么写入太多数据（吞吐太高）
       + 网络使用率
         ```
         nicstat 5
         ```
         千兆位网络理论每秒处理125MB，但网络无法支持100%使用率，对于本地以太局域网来说，
         承受的网络使用率超过40%就以为着接口饱和了，所以基于网络的应用，要监控网络确保其不成为瓶颈。
   
   + java 监控工具
    
        + **jcmd** 用于打印java进程所涉及的基本类，线程和VM信息
        + **jconsole** 提供JVM活动的图形化师徒，包括线程和类的使用以及GC活动
        + **jhat** 读取分析内存堆转储
        + **jmap** 提供堆转储和其他JVM内存使用的信息
        + **jinfo** 查看JVM系统属性，可以动态设置一些系统属性
        + **jstack** 转储栈信息
        + **jstat** 提供GC和类装载活动的信息
        + **jvisualvm** 监视JVM的GUI工具
        + **Arthas** 阿里巴巴开源java诊断工具
    
## JIT编译器

     JIT是just in time 的缩写, 也就是即时编译编译器。使用即时编译器技术，能够加速 Java 程序的执行速度。
     对于 Java 代码，刚开始都是被编译器编译成字节码文件，然后字节码文件会被交由 JVM 解释执行，所以可以说 Java 本身是一种半编译半解释执行的语言。
     Hot Spot VM 采用了 JIT compile 技术，将运行频率很高的字节码直接编译为机器指令执行以提高性能，所以当字节码被 JIT 编译为机器码的时候，要说它是编译执行的也可以。也就是说，运行时，部分代码可能由 JIT 翻译为目标机器指令（以 method 为翻译单位，还会保存起来，第二次执行就不用翻译了）直接执行。

   + **client**   -server
        
     client编译器开启编译比server编译器要早，适用于注重启动时间的应用
     
   + **server**  -client
     
        适用于长期运行又要求高性能的应用
        

   
   + 分层编译   -XX:TieredCompilation
     
        启动时用client编译器，代码变热后用server编译器
       
   + 调优代码缓存
        
        JVM编译代码后会保留编译后的汇编语言指令集，代码缓存的大小固定，如果代码缓存过小，一旦被热点代码占满了空间，会导致其他代码解释执行
        
        设置代码缓存最大值：**-XX:ReservedCodeCacheSize=N**
        设置代码缓存初始值：**-XX:InitialCodeCacheSize=N** 
     
   + 编译阈值 
     
     jvm通过方法调用计数器和回边计数器检测方法调用总数，达到阈值是，开启编译
     
     **-XX:CompileThreshold=N** 设置代码循环执行多少次转而进行编译，调整这个值可以使编译更早发生
     

   + 检测编译过程
     -XX:PrintCompilation
   
   + 编译线程
        
     当方法（或循环）适合编译时，会进入编译队列，调用次数最高的方法拥有最高优先级，编译队列则由一个或多个后台进程处理

## 垃圾回收器
   对于java而言垃圾回收主要分为两步

    1.找到不再使用的垃圾对象
    2.释放这些对象所占用的内存（必要时还需要进行整理）
  
   分代算法内存模型
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/heapstruct.png)

   标记整理(mark-sweep-compact)算法示例
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/GC_Mark_Compact.jpg)
    
 
   HotSpot虚拟机所包含的常见垃圾回收器,**如果两个收集器之间存在连线，则说明它们可以搭配使用**

   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%99%A8.png)
   
   | 垃圾回收算法|简介|特点|适用场景|
   | :----| :---- |:---- |:---- |
   | Serial 收集器| Serial收集器是最基本的、发展历史最悠久的收集器 |单线程、简单高效（与其他收集器的单线程相比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程手机效率。收集器进行垃圾回收时，必须暂停其他所有的工作线程，直到它结束（Stop The World）|适用于Client模式下的虚拟机，100M以下堆 
   | ParNew收集器| Serial收集器的多线程版本，除了使用多线程外其余行为均和Serial收集器一模一样 |多线程、ParNew收集器默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境中，可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数。和Serial收集器一样存在Stop The World问题 |ParNew收集器是许多运行在Server模式下的虚拟机中首选的新生代收集器，因为它是除了Serial收集器外，唯一一个能与CMS收集器配合工作的。
   | Parallel Scavenge 收集器| 与吞吐量关系密切，故也称为吞吐量优先收集器。 |属于新生代收集器也是采用复制算法的收集器，又是并行的多线程收集器;GC自适应调节策略（与ParNew收集器最重要的一个区别） |注重高吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge+Parallel Old 收集器
   | Serial Old 收集器| Serial Old是Serial收集器的老年代版本。 |样是单线程收集器，采用标记-整理算法。|主要也是使用在Client模式下的虚拟机中。也可在Server模式下使用。
   | Parallel Old 收集器| Parallel Scavenge收集器的老年代版本。 |多线程，采用标记-整理算法 |注重高吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge+Parallel Old 收集器 
   | CMS收集器| 一种以获取最短回收停顿时间为目标的收集器。 |基于标记-清除算法实现。并发收集、低停顿。|适用于注重服务的响应速度，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务。
   | G1收集器| 一款面向服务端应用的垃圾收集器 |并行与并发：使用多个CPU来缩短Stop-The-World停顿时间，支持用户线程和回GC线程并发执行。分代收集：G1能够独自管理整个Java堆，并且采用不同的方式去处理不同存活周期的对象；空间整合：G1运作期间不会产生空间碎片，收集后能提供规整的可用内存。可预测的停顿：G1除了追求低停顿外，还能建立可预测的停顿时间模型。|CMS适用算法较G1简单，在小堆是性能更佳；而在大堆中，G1性能更佳；CMS不对堆进行压缩整理，碎片率较G1高

### GC调优基础
   + 调整堆的大小  -Xms -Xmx
   + 代空间的调整  调整新老年代空间的大小；新生代过大垃圾收集频率低，但相应老年代太小易引发Full Gc
   + 控制并发    调整GC线程数，__ParallelGCThreads = 8+((N-8)*5/8)__，如果多个JVM运行在同一台物理机，需要减少该线程数
   + 自适应调整   默认开启，JVM会根据策略，不断尝试选择最优代空间分配
   
###CMS垃圾收集器调优
   
   CMS收集器的工作过程图：

   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/CMS%E6%94%B6%E9%9B%86%E8%BF%87%E7%A8%8B.png)
   
   CMS收集器有三种基本操作

   + CMS会对新生代的对象进行回收
   + CMS会启动一个并发的线程对老年代进行回收 
   + 如果有必要，CMS会发起Full Gc
     + **并发模式失效**： 新生代发生垃圾回收，同时老年代又没有足够的空间容纳晋升对象时，CMS垃圾回收会退化成FullGc
     + **晋升失败**： 老年代有足够空间容纳晋升的对象，但由于空闲空间过于碎片化，晋升失败
     + **解决方案**
        + 增大堆空间or增大老年代空间  **MaxGCPauseMills=N,GCTimeRatio=N**
        + 以更高的频率运行后台回收线程  **-XX:CMSInitiatingOccupancyFraction=N -XX:+CMSInitiatingOccupancyOnly**
        + 使用更多的后台回收线程   **XX:ConcGCThreads=N**
    
### G1垃圾回收调优
   G1保留分代思想，但它是针对整个堆并以Region为单位进行垃圾回收，G1的分块大体结果如下图所示：
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/G1Region.png)

 
   G1收集器运行示意图：
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/G1.png)

   G1提供了两种GC模式，**Young GC**和**Mixed GC**

   **Young GC**：选定所有年轻代里的Region。通过控制年轻代的Region个数，既年轻代内存大小，来控制Young GC的时间开销。
   
   **Mixed GC**：选定所有年轻代里的Region，以及并发标记阶段统计得出收集收益高的若干老年代Region。在用户指定的开销目标范围内尽可能选择收益高的老年代Region。
   
   G1 只会回收垃圾最多的分区，主要包括以下四种操作
   + 新生代垃圾回收
   + 后台收集，并发周期
   + 混合式垃圾收集
   + 必要时的Full GC
     + **并发模式失效** ：G1垃圾收集启动标记周期，但老年代在周期完成之前已被填满
     + **晋升失败**： G1收集器完成标记阶段，开启混合式垃圾回收，清理老年代的分区，不过老年代空间在垃圾回收释放出足够内存之前被耗尽
     + **疏散失败** ： 进行新生代垃圾收集时，Survivor空间和老年代中没有足够的空间容纳所有的幸存对象
     + **大对象分配失败**： 分配大对象可能导致Full Gc
     + **解决方案**
        + 增加堆大小或调大老年代
        + 增加后台GC线程数 __ConcGCThreads = (ParallelGCThreads+2)/64__
        + 以更高的频率进行后台垃圾收集 __-XX:CMSInitiatingHeapOccupancyPercent=N__ 设置G1在达到N比率时启动收集周期
        + 在混合式垃圾回收周期中完成更多的垃圾收集工作
          
           + **G1MixedGCLiveThresholdPercent**：老年代中的存活对象的占比，只有在此参数之下，才会被选入CSet（老年代收集集合）。
           + **G1MixedGCCountTarget**：一次并发标记之后，最多执行Mixed GC的次数。
           + **G1OldCSetRegionThresholdPercent**：一次混合 GC中能被选入CSet的最多老年代Region数量。
           + **MaxGCPauseMills** 调节GC停顿最大可忍受时长，调大则每次混合式GC能收集更多老年代分区 
    
###GC高级调优
    
   + 晋升及Survivor空间
        + Survivor空间太小，被占满后，新生代的活跃对象被迫移步老年代   
          如果老年代GC频繁，可以尝试调大堆大小或新生代大小（survivor根据新生代大小按比例算出）
        + 对象在Survivor中经历的GC周期数有上限，超过则被移动到老年代
          如果要避免对象过早晋升，可以调大阈值
          
   + 分配大对象       
        
      ”大型“是一个相对的概念，它取决于Eden空间内TLAB(线程本地分配缓冲区)；
       TLAB是线程固有的Eden内存空间，避免了同步操作; 但TLAB过小会导致程序在TLAB之外分配对象，降低性能；因此当需要大量分配大型对象时，调大TLAB大小或新生代大小能提升性能
   + 巨型对象
       TLAB无法分配的对象，JVM会尝试在Eden中分配，如果仍无法分配，将直接进入老年代；在G1中如果对象超过Region大小，将进入老年代；如若老年代没有合适的连续空间，则可能被迫Full GC，这种情况下需要增大Region的大小 
     

## 堆内存调优
    有节制的创建对象并尽快丢弃，使用更少内存，能显著提升垃圾收集器性能
    
   + 减少内存使用 
     + 减少对象大小
        
       减少实例变量个数，
       能用int 就不用long，能用float就不用double;
       
     + 延迟初始化
        
        ```
        public class SafeDoubleCheckedLocking {
        private volatile static Instance instance;

        public static Instance getInstance() {
            if (instance == null) {
                synchronized (SafeDoubleCheckedLocking.class) {
                    if (instance == null)
                        instance = new Instance();//instance为volatile
                }
            }
            return instance;
        }
        }
        ```
       
        + 尽早清理
          ```
            public E remove(int index) {
                rangeCheck(index);
            
                modCount++;
                E oldValue = elementData(index);
            
                int numMoved = size - index - 1;
                if (numMoved > 0)
                System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
                elementData[--size] = null; // 清理 让GC完成其工作
            
                return oldValue;
            }
          ```
        
     + 字符串保留
            
        对于同个字符序列，没有必要存在多个字符串表示;
   
   + 对象生命周期管理
    
     + 对象重用
       
        + 对象池（线程池，JDBC池。。。）
        + 线程局部变量 （ThreadLocal）


## 原生内存调优
    
   + 大页
        
        ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/pagetable.png)
     
        视操作系统而言，一般默认开启
     
        **页**：页是操作系统内存分配的最小单位，要分配1字节，必须分配1个整页，直至被分配完毕，才会分配一个新页
        
        **页表**：存放逻辑页与物理页帧的对应关系。 每一个进程都拥有一个自己的页表
        
        **TLB**: 最常用的页表映射保存在TLB中，即页表缓存，但其数目有限；故如果每个页能表示更多内存，则只需要更少的TLB表项就能覆盖 整个程序的内存，查找性能更佳
   + 压缩的oop
    
        在32位系统中对象引用占**4个字节**，64位系统中占**8个字节**，意味着需要更多的GC周期，因为堆中留给其他数据的空间变少了；
        
        JVM可以使用压缩的oop来减少额外的内存损耗  **-XX:-UseCompressedOops**
        ```
        Integer i = new Integer(23);
        ```
        
        ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/oop.png)
        
        启用压缩oop后，对象指针变为了4字节
     
        + 对象指针的实现
            
          在oop只有32位长是，只可以引用2^32 = 4GB;
          固采用一种折中方案，使用**35位**的oop，2^35=32GB,在进入寄存器时左移三位，从寄存器读出时右移三位
          

## 线程与同步的调优
    
   + 线程池 
        
        线程池大小对获取线程池最佳性能至关重要，过大则带来上下文切换开销，过小增加任务阻塞时长
        ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/TreadPool.png)
     
        线程池动态化配置方案
        ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/dynamicthreadpool.png)
   
   + 线程同步
        
       + 锁定对象的开销
         
          + 重量级锁在多线程竞争情况下，对于每个线程的开销都是一样的，都必须执行加锁操作
          + CAS很依赖CPU高速缓存，存在CACHE Miss情况，在多线程竞争时，开销无法预测，最极端的情况下，两个线程都看到对方同时在修改变量值，可能出现死循环。
          ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/cache_miss.png)  
       + 避免同步
         
          在不存在竞争或少量竞争的情况下，CAS 优于synchronized
          ；在只读不写的情况下，CAS 优于synchronized；
          竞争剧烈的情况下，synchronized表现更好
         
   + 伪共享
     
        ```
            pulic class DataHolder{
                public volatile long l1;
                public volatile long l2;
                public volatile long l3;
                public volatile long l4;
            }
        ```
        基于局部性原理，CPU可能将 l1,l2 加载至同一缓存行，当该缓存行被更新，当前核必须通知其他所有核这个内存被修改了
        
   + JVM线程调优
     
     + 调节线程栈大小
       
          64位系统线程栈默认大小256kb，在内存比较稀缺的机器上可以减少线程栈大小
     + **偏向锁**
       
        java1.6后默认开启，当竞争激烈时偏向锁意义不大，且增大了开销
       

## 其他调优
    
   + **transient**字段，减少需要序列化的数据
   + 预估设定集合的大小，避免扩容
   + 

    
    