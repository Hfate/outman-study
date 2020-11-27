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