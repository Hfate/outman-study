# 概述
    redis是一个开源的，由C语言编写的，非关系型内存数据库，其特点是高性能，常见应用场景是缓存；
    
# 数据结构
   
   ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/redis-data-structure-types.jpeg)

   ## 简单动态字符串
   + **简介** 
   
         字符串是Redis中最为常见的数据存储类型，其底层实现是简单动态字符串sds(simple dynamic string)，是可以修改的字符串。
   + **定义**:
        ```
            每个sds.h/sdshdr结构表示一个SDS值
     
            struct sdshdr{
                //记录buf数组中已使用字节的数量
                //等于SDS所保存字符串的长度
                int len;
                //记录buf数组中未使用字节的数量
                int free;
                //柔性数组，不占用内存，用于保存字符串
                char buf[];
            }
        ```
   + **示例**
   
       ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/1301290-20190420105230080-1708767435.png)
       + free属性的值为5，表示空闲空间长度为5
       + len的属性为5，表示这个SDS保存了一个五字节长的字符串
       + buf属性是一个char类型的数组，数组的最后一个字节保存了空字符'\0',SDS遵循C字符的以空字符串结尾的惯例，这一字节空间不计算在len属性里
            
   + SDS与C字符串的区别
   
        + len属性记录了SDS本身的长度，常数复杂度获取字符串长度
        + 杜绝缓冲区溢出，遭遇修改操作时，API会先检查SDS是否满足修改需求，不满足的话会自动扩容
        + 减少修改带来的内存重分配次数
            + 空间预分配
                 ```
                n<1M    2n  
                n>=1m   n+1m    
                ```
            +   惰性空间释放
            
                当SDS的API需要缩短SDS所保存的字符串时，程序并不立即使用内存重分配来回收收缩后多出来的字节，而使用free属性将这些字节的数量记录下来   
   + 二进制安全
   
       + C字符串必须符合某种编码，且以空字符串判定结尾，不适用与图片，音视频等等
       + SDS以buf保存字节数组，以len属性判定结尾，适用任意格式二进制数据
   + 总结

        | C字符串 | SDS | 
        | :-----| :---- | 
        | 获取字符串长度的复杂度为O(n) | 获取字符串长度的复杂度为O(1) | 
        | API是不安全的，可能造成缓冲区溢出 | API是安全的，不会造成缓冲区溢出 |
        | 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次，最多需要执行N次内存重分配 | 
        | 只能保存文本数据 | 可以保存文本或任意二进制数据 | 
        | 可以使用所有<string.h>库中的函数 | 可以使用一部分<string.h>库中的函数 |   



   ## 链表
   + 简介
   
    链表提供高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度；
    常见应用比如列表键，发布与订阅，慢查询，监视器等功能都用到了链表；
   + 定义
    
        ```
            typedef struct listNode{
                // 前置节点
                struct listNode *prev;
               // 后置节点
                struct listNode *next;
               // 节点的值
                void *value;
            }listNode;
        ```
        多个listNode可以通过prev和next指针组成双端链表
        ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/1585877706495874.jpg)
        但使用adlist.h/list来持有链表的话，操作会更方便
        ```
            typedef struct list{
                // 表头节点
                listNode *head;
                // 表尾节点
                listNode *tail;
                // 链表锁包含的节点数量
                unsigned long len;
                // 节点值复制函数
                void *(*dup)(void *ptr);
                // 节点值释放函数
                void (*free)(void *ptr);
                // 节点值对比函数
               int (*match)(void *ptr,void *key);
            }
        ```
        ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/v2-b2149485295efb359a851354776759b7_720w.jpg)

        
   + 总结
      + 双端： 链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1);
      + 无环： 表头节点的prev指针和表尾节点的next指针都指向null，对链表的访问以null为终点
      + 带表头的指针和表尾指针：通过list结构的head指针和tail指针，获取复杂度为O(1);
      + 带链表长度计算器：list结构的len属性对list持有的节点数进行计数，获取复杂度为O(1);
      + 多态： 链表节点使用void*指针来保存节点值，并且可以通过list结构的dup，free，match三个属性为节点设置类型特定函数，所以链表可以保存不同类型的值
      
      
      
   ## 字典
   + 简介
    
    字典，又称符号表，关联数组或映射（map），是一种保存键值对的抽象数据结构。
    redis的数据库就是使用字典作为底层实现的，对数据库的增删改查都是建立在对字典的操作之上。
    
   + 字典的实现
      + 哈希表
           ```
            typedef struct dictht{
                //哈希表数组
                dictEntry **table;
                
                //哈希表大小
                unsigned long size;
        
                // 哈希表大小掩码，用于计算索引值
                // 总是等于size-1
                unsigned long sizemark;
                
                // 该哈希表的已有节点的数量
                unsigned long used;
            } dictht;
           ```   
      
      + 哈希表节点
     
           哈希表节点使用dictEntry结构表示，每个dictEntry结构都保存着一个键值对
          ```
              typedef struct dictEntry{
                // 键
                void *key;
                // 值
                union{
                    void * val;
                    unit64_t u64;
                    int64_t s64;
                } v;
                //指向下个哈希表节点，形成链表
                struct dictEntry *next;
              }  dictEntry
          ```
          ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/20190314145831191.jpg)   
      
      + 字典
      
        redis中点字典由dict.h/dict结构表示
        ```
            typedef struct dict{
                //类型特定函数,每个dictType保存了一簇用于操作特定类型键值对的函数。redis会为用途不同的字典设置不同的类型特定函数
                dictType * type;
                // 私有数据，保存了需要传给类型特定函数的参数
                void * privdata;
                //哈希表 一般使用h[0],h[1]只会在对h[0]rehash时使用
                dictht ht[2];
        
                //rehash索引
                // 当rehash不在进行时，值为-1；
                int trehasidx;
            } dict;
        ```
        
      ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/1335795-20191014200059462-1040567548.png)
      
      + 哈希算法
                
                MurmurHash2算法
      + 解决键冲突
        
                链地址法，每个哈希表节点都有一个next指针，多个哈希表节点可以构成一个单向链表
      + rehash
        +  为字典的ht[1]哈希表分配空间，这个哈希表的空间的大小取决于要执行的操作，以及ht[0]当前包含的键值对的数量
            + 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2^n
            + 如果是收缩操作，那么ht[1]的大小为第一个等于ht[0].used*2^n
        + 将保存在ht[0]中所有的键值对rehash到ht[1]上面
        + 释放ht[0],与ht[1]位置交换
      + 渐进式rehash
        + 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
        + 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始
        + 在rehash期间，每次对字典增删改查，程序顺带将ht[0]哈希表在rehasidx索引上的所有键值对rehash到ht[1]；
        + 直至渐进式rehash完成，rehashidx等于-1     
        + 在渐进式rehash过程中，如果请求在ht[0]未找到，将继续去ht[1]中去找；
        
        
   ## 跳跃表
        
   + 简介
    
         跳跃表是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。跳跃表支持平均O（logN），最坏O(N)复杂度的节点差照片，还可以通过顺序性操作批量处理节点；
   Redis的跳跃表由redis.h/zskiplistNode和redis.h/zskiplist 两个结构定义，其中zskiplistNode结构用于表示跳跃表节点，而zskiplist结构则用于保存跳跃表节点的相关信息，比如节点的数量，以及指向表头节点和表尾节点的指针
   + zskiplist
        + header 指向跳跃表的表头节点
        + tail 指向跳跃表的表尾节点
        + level 记录跳跃表内，层数最大的那个节点的层数
        + length 记录跳跃表的长度，即目前跳跃表内包含的节点数 
   跳跃表节点的实现由redis.h/zskiplistNode结构定义
   ```
        typedef struct zskiplistNode{
            // 后退指针
            struct zskiplistNode *backward;
            // 分值
            double score;
            //成员对象
            robj *obj;
            
            //层
            struct zskiplistLevel{
                //前进指针
                struct zskiplistNode *forward;
                //跨度
                unsined int span;
            } level[];
        } zskiplistNode;
   ```    
    
   ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/Redis%E8%B7%B3%E8%B7%83%E8%A1%A8.png)   
            
   + 层：每个层高是1-31之间的随机数，节点中用L1，L2，L3等字样标记各个层，L1代表第一层，以此类推；每个层都带有两个属性:前进指针和跨度。前进指针用于访问位于表尾方向的其他节点，
   而跨度则记录了前进指针所指向节点和当前节点的距离。在上面的图片中，连线上带有数字的箭头就代表前进指针，而那个数字就是跨度。当程序从表头向表尾进行遍历时，访问会沿着层的前进指针进行。
   
   + 后退指针：节点中用BW字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。
   + 分值: 图中节点中的1.0,2.0指分值，各节点按分值大小排序，分数相同按对象大小排序；多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的。
   + 成员对象：o1,o2是节点保存的成员对象
   

## 整数集合
   + 简介
   
            整数集合是集合键的底层实现之一，当一个集合只包含整数值元素时，并且这个集合的元素不多时，Redis就会使用整数集合作为集合键的底层实现
        
   + 实现
        每个intset.h/intset结构表示一个整数集合
   ```
        typedef struct intset{
            //编码方式,表示contents数组中包含元素的整数类型 例如 int16  int32 int64
            uint32_t encoding;
            // 集合包含的元素数量
            uint32_t length;
            //保存元素的数组
            int8_t contents[];
        } intset;
   ```        

   + 升级
      如果一直往整数集合里添加int16_t类型的值，那么数组一直是int16_t类型，当放入int32_t时，程序会对数组进行升级。升级后不会降级。
      

## 压缩列表
  + 简介 
            
            压缩列表是Redis为了节约内存而开发的,是由一系列特殊编码的连续内存块组成的顺序型数据结构；一个压缩列表的可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值。
            压缩列表时列表键与哈希键的底层实现之一。当一个列表键值包含少量列表项，并且每个列表项都要么是小整数值，要么就是长度较短的字符串。，那么就会用压缩列表实现      
           
     
  + 数组的优势在于占用一片连续的空间可以很好的利用CPU缓存的优势；而常规数组要求每个元素大小相同，那么对于存储不规则的数据集合，难免会造成空间的浪费.
  ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/%E6%95%B0%E7%BB%84.png)
        
  + 那么势必会想到对数组进行压缩，同时为了准确识别单个元素，增加一个length属性，这便是简单的压缩链表的一个算法思想；
  ![avatar](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8.png)

  
  + 压缩列表各个组成部分详细说明
     
  | 属性 | 类型 | 长度|用途 
  | :-----| :---- | :---- |:---- |
  | zlbytes | uint32_t | 4字节 |记录整个压缩列表占用字节数；在对压缩列表进行内存重分配，或者计算zlend的位置时使用 |
  | ztail | uint32_t |4字节 |记录压缩列表表尾节点距离压缩列表的其实地址有多少字节；通过这个偏移量，程序无需遍历整个压缩列表就可以确定表节点的地址 |
  | zllen | uint16_t |2字节 |记录了压缩列表包含的节点数量；当这个属性的值大于UNIT16_MAX时，节点的真是数量需要遍历整个压缩列表才能计算出 | 
  | entryX | 列表节点 | 不定 |压缩列表包含的各个节点，节点的长度由接地那保存的内容决定 |
  | zlend | uint8_t | 1字节 |特殊值0xFF(十进制255),用于标记压缩列表的末端 |      
  
## 对象
                      
                