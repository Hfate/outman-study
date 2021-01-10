# Zab协议
    Zab协议 的全称是 Zookeeper Atomic Broadcast （Zookeeper原子广播）。
    Zookeeper 是通过 Zab 协议来保证分布式事务的最终一致性。
    
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/1609264542(1).jpg) 
   + Zab协议是为分布式协调服务Zookeeper专门设计的一种 支持崩溃恢复 的 原子广播协议 ，是Zookeeper保证数据一致性的核心算法。Zab借鉴了Paxos算法，但又不像Paxos那样，是一种通用的分布式一致性算法。它是特别为Zookeeper设计的支持崩溃恢复的原子广播协议。
   + 在Zookeeper中主要依赖Zab协议来实现数据一致性，基于该协议，zk实现了一种主备模型（即Leader和Follower模型）的系统架构来保证集群中各个副本之间数据的一致性。
这里的主备系统架构模型，就是指只有一台客户端（Leader）负责处理外部的写事务请求，然后Leader客户端将数据同步到其他Follower节点。
   + Zookeeper 客户端会随机的链接到 zookeeper 集群中的一个节点，如果是读请求，就直接从当前节点中读取数据；如果是写请求，那么节点就会向 Leader 提交事务，Leader 接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交。

   + Zab 协议的特性：
    
         1）Zab 协议需要确保那些已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交。  
         2）Zab 协议需要确保丢弃那些只在 Leader 上被提出而没有被提交的事务。

   + Zab 协议实现的作用
    
        + 使用一个单一的主进程（Leader）来接收并处理客户端的事务请求（也就是写请求），并采用了Zab的原子广播协议，将服务器数据的状态变更以 事务proposal （事务提议）的形式广播到所有的副本（Follower）进程上去。
        + 保证一个全局的变更序列被顺序引用。
                
              Zookeeper是一个树形结构，很多操作都要先检查才能确定是否可以执行，比如P1的事务t1可能是创建节点"/a"，t2可能是创建节点"/a/bb"，只有先创建了父节点"/a"，才能创建子节点"/a/b"。
              为了保证这一点，Zab要保证同一个Leader发起的事务要按顺序被apply，同时还要保证只有先前Leader的事务被apply之后，新选举出来的Leader才能再次发起事务。

        + 当主进程出现异常的时候，整个zk集群依旧能正常工作。

   + Zab 协议如何保证数据一致性
   
     假设两种异常情况：
   
        + 一个事务在 Leader 上提交了，并且过半的 Folower 都响应 Ack 了，但是 Leader 在 Commit 消息发出之前挂了。
        + 假设一个事务在 Leader 提出之后，Leader 挂了。
    
      要确保如果发生上述两种情况，数据还能保持一致性，那么 Zab 协议选举算法必须满足以下要求：
    
      Zab 协议崩溃恢复要求满足以下两个要求：
        
        + 确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交。
        + 确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal。
根据上述要求
Zab协议需要保证选举出来的Leader需要满足以下条件：
1）新选举出来的 Leader 不能包含未提交的 Proposal 。
即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。
2）新选举的 Leader 节点中含有最大的 zxid 。
这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。

Zab 的四个阶段
选举阶段
Fast Leader Election（快速选举）
前面提到的 FLE 会选举拥有最新Proposal history （lastZxid最大）的节点作为 Leader，这样就省去了发现最新提议的步骤。这是基于拥有最新提议的节点也拥有最新的提交记录
成为 Leader 的条件：
1）选 epoch 最大的
2）若 epoch 相等，选 zxid 最大的
3）若 epoch 和 zxid 相等，选择 server_id 最大的（zoo.cfg中的myid）

发现阶段
这个阶段的主要目的是发现当前大多数节点接收的最新 Proposal，并且准 Leader 生成新的 epoch ，让 Followers 接收，更新它们的 acceptedEpoch

同步阶段
同步阶段主要是利用 Leader 前一阶段获得的最新 Proposal 历史，同步集群中所有的副本。

广播阶段（Broadcast）
到了这个阶段，Zookeeper 集群才能正式对外提供事务服务，并且 Leader 可以进行消息广播。同时，如果有新的节点加入，还需要对新节点进行同步。