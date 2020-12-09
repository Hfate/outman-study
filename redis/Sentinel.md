# Sentinel
   + 简介
    
    sentinel是redis的高可用解决方案，有一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主从服务器；
    从代码实现层面讲，sentinel本质上是一个运行在特殊模式下的Redis服务器
    
   + 初始化Sentinel状态
        ```
            struct sentinelState{
                //保存了所有被这个sentinel监视的主服务器
                // 字典的键是一个指向sentinelRedisInstance结构的指针
                dict *masters;
                
                // 是否进入了TILT模式
                int tilt;
                //目前正在执行的脚本的数量
                int  running_scripts;
                // 进入TILT模式的时间
                mstime_t tilt_start_time;
                //最后一次执行事件处理器的时间
                mstime_t previous_time;
                // 一个FIFO 队列
                包含了所有需要执行用户的脚本
                list *scripts_queue;
            } sentinel;
        ``` 
   + 初始化Sentinel状态masters属性
        + 字典的键时被监视的主服务器的名字
        + 字典的值时被监视主服务器对应的sentinel.c/sentinelRedisInstance结构
     ```
        typedef struct sentinelRedisInsttance{
            //标识值
            int flags;
            //实例的名字
            // 主服务器的名字由用户配置文件中设置
            // 从服务器以及Sentinel的名字由Sentinel 自动设置
            // 格式为ip:port  例如“127.0.0.1:26379”
            char *name;
     
            //实例的运行ID
            char *runid;
            
            //配置纪元，用于实现故障转移
            unit64_t config_epoch;
     
            //实例的地址
            sentinelAddr *addr;
            
            //sentinel down-after-millseconds 选项的值
            //实例无响应多少毫秒之后才会被判断为主观下线
            mstime_t down_after_period;
     
            // 判断这个实例为客观下线所需支持投的票数
            int quorum；
     
            // 执行故障转移时，可以同时对新的主服务器进行同步的从服务器数量
            int parallel_syncs;
            
            // 刷新故障迁移状态的最大时限
            mstime_ failover_timeout;
     
        } 
     ```
   + 创建连向主服务器的网络连接
    
        初始化sentinel后时创建连向主服务器器的两个连接，sentinel将成为主服务器的客户端。
        + 命令连接，用户发送和接收命令回复
        + 订阅连接  订阅主服务器的_sentinel_:hello频道
        
   + 获取主服务器的信息
   
        每十秒向主服务器发送一次INFO命令，并通过分析回复来获取主服务器当前信息
        + 服务器本身的信息，服务器id和角色信息
        + 从服务器的信息;ip+port;
        
        
   + 获取从服务器的信息
        
        当发现主服务器由新的从服务器出现时，sentinel会向新的从服务器发送两个连接，并每隔十秒发送一次INFO命令，获取回复；
        + 从服务器运行的ID
        + 从服务器的角色
        + 主服务器的ip+port
        + 主从服务器的连接状态
        + 从服务器的优先级
        + 从服务器的复制偏移量
        
   + 向主服务器和从服务器发送信息
        
        sentinel每隔两秒会向所有被监视的主服务器和从服务器发送命令
        ```
            PUBLISH _SENTINEL_:HELLO "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name><m_ip>,<m_port>,<m_epoch>
        ```   
   
   + 接收来自主服务器和从服务器的频道信息
        sentinel不仅向主从服务器的_sentinel_:hello 频道发送命令，并且订阅该频道的消息，用以sentinel集群之间的相互感知
        
   
   + 更新sentinel字典
        sentinel从_sentinel_:hello 频道获取到其他sentinel实例的信息，更新或保存在自己本地的sentinel字典中
        
   + 创建连接其他sentinel的命令连接
        最终监听同一个主服务器的多个sentinel会形成一个相互连接的网络
        
        
   + 检测主观下线状态
   
        sentinel每秒一次的频率向所有与它创建连接的实例发送ping命令，并可能得到不同回复
        + 有效回复  
             
             实例返回+PONG,-LOADING,-MASTERDOWN中一种
        + 无效回复
              
             实例返回+PONG,-LOADING，-MASTERDOWN三种之外其他回复，或指定时限内（默认五十秒）没有回复，则会被判定为主观下线
             
             
   + 检查客观下线状态
        
       当sentinel将一个主服务器判断为主观下线后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他的sentinel进行询问，
       看它们是否也认为主服务器进入下线状态，当sentinel从其他sentinel那收到足够数量的下线判断后，sentinel将会将从服务器判定为客观下线，
       并对主服务器执行故障转移操作           
    
   + 选举领头Sentinel 
    
        + 所有在线的sentinel都有被选为领头sentinel的资格，换句话说，
          监视同一个主服务器的多个在线sentinel中的任意一个都有可能成为领头的sentinel；
        + 每次进行领头sentinel在选举之后，不论选举是否成功 ，所有sentinel的配置纪元的值都会自增一次。
        + 在一个配置纪元丽，所有sentinel都有一次将某个sentinel设置为局部领头sentinel的机会，
           并且局部领头一旦设置，在这个配置纪元里面就不能再更改。
        + 每个发现主服务器进入客观下线的sentinel都会要求其他sentinel将自己设置为局部领头sentinel。
        + 当一个sentinel（源）向另一个sentinel（目标）发送SENTINEL-is-master-down-by-addr命令，
          并且命令中的runid参数不是*符合而是源sentinel的运行ID时，这表示源Sentinel要求目标sentinel将前者设置为自己的局部领头sentinel；
        + sentinel设置局部领头的规则是先到先得；最先向目标sentinel发送设置要求的源sentinel将成为目标sentinel的局部领头sentinel，
          而之后接收到的所有设置要求都会被目标sentinel拒绝。
        + 目标sentinel在接收到SENTINEL is-master-down-by-addr命令之后，将向源Sentinel返回一条命令回复，回复中包含leader_runid参数和leader_epoch参数；
        + 源sentinel在接收到目标sentinel返回的命令回复后，会检查回复中leader_epoch参数的值和自己的配置纪元是否相同，
          如果相同的话，那么源sentinel继续取出回复中的leader_id参数，如果leader_runid参数的值和源sentinel运行ID一致，那么表示目标sentinel将源sentinel设置为局部领头sentinel。
        + 如果有某个sentinel被半数以上的sentinel设置成局部领头sentinel，那么这个sentinel成为领头sentinel  
        + 因为领头sentinel的产生需要半数以上的sentinel的支持，并且每个sentinel在每个配置纪元里面只能设置一次局部领头sentinel，所以在一个配置纪元中， 只会出现一个领头sentinel；
        + 如果给定时限内，没有一个sentinel被选举成为领头sentinel，那么各个sentinel将在一段时间之后再次进行选举，直到选出领头sentinel为止。  
                  
              
   + 故障转移
        
        + 在已下线主服务器树下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器
        + 让已下线的主服务器属下的所有从服务器复制新的主服务器
        + 将已经下线的主服务器设置为新的主服务器的从服务器
        
                   
        
                      
             
             
          
          
    