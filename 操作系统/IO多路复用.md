# IO多路复用
    I/O multiplexing 也就是我们所说的I/O多路复用，
    那么字面上来看I/O multiplexing 就是将多个I/O凑在一起。
    就像下面这张图的前半部分一样，中间的那条线就是我们的单个线程，它通过记录传入的每一个I/O流的状态来同时管理多个IO。
   
   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/io%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8.png)
  + I/O多路复用的实现
       ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/io%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A82.png)
       + select 
       
            + 当进程调用select，进程就会被阻塞
            + 此时内核会监视所有select负责的的socket，当socket的数据准备好后，就立即返回。
            + 进程再调用read操作，数据就会从内核拷贝到进程。  
            + 伪代码实现
                 ```
                    int s = socket(AF_INET, SOCK_STREAM, 0);   
                    bind(s, ...) 
                    listen(s, ...) 
                     
                    int fds[] =  存放需要监听的socket 
                    
                      while(1){ 
                         int n = select(..., fds, ...) 
                        for(int i=0; i < fds.count; i++){ 
                            if(FD_ISSET(fds[i], ...)){ 
                                //fds[i]的数据处理 
                            } 
                        } 
                 ```
            + select函数的调用过程
              
                 + 从用户空间将fd_set拷贝到内核空间
                 + 注册回调函数
                 + 调用其对应的poll方法
                 + poll方法会返回一个描述读写是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。
                 + 如果遍历完所有的fd都没有返回一个可读写的mask掩码，就会让select的进程进入休眠模式，直到发现可读写的资源后，重新唤醒等待队列上休眠的进程。如果在规定时间内都没有唤醒休眠进程，那么进程会被唤醒重新获得CPU，再去遍历一次fd。
                 + 将fd_set从内核空间拷贝到用户空间
              
            + select函数优缺点
              
                  缺点：两次拷贝耗时、轮询所有fd耗时，支持的文件描述符太小
                  优点：跨平台支持  
       + poll
       
           + poll函数的调用过程（与select完全一致）
           + poll函数优缺点
           
                 优点：连接数（也就是文件描述符）没有限制（链表存储）
                 缺点：大量拷贝，水平触发（当报告了fd没有被处理，会重复报告，很耗性能）
       + epoll
            + 伪代码实现
                  ```
                     int s = socket(AF_INET, SOCK_STREAM, 0);    
                     bind(s, ...) 
                     listen(s, ...) 
                      
                     int epfd = epoll_create(...); 
                     epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中 
                      
                     while(1){ 
                         int n = epoll_wait(...) 
                         for(接收到数据的socket){ 
                             //处理 
                         } 
                     } 
                 ```
            + epoll的ET与LT模式
            
                  LT：延迟处理，当检测到描述符事件通知应用程序，应用程序不立即处理该事件。那么下次会再次通知应用程序此事件。
                  ET：立即处理，当检测到描述符事件通知应用程序，应用程序会立即处理。
                  ET模式减少了epoll被重复触发的次数，效率比LT高。我们在使用ET的时候，必须采用非阻塞套接口，避免某文件句柄在阻塞读或阻塞写的时候将其他文件描述符的任务饿死  
                  
            + 水平触发和边缘触发模式区别
            
                + 读缓冲区刚开始是空的
                + 读缓冲区写入2KB数据
                + 水平触发和边缘触发模式此时都会发出可读信号
                + 收到信号通知后，读取了1kb的数据，读缓冲区还剩余1KB数据
                + 水平触发会再次进行通知，而边缘触发不会再进行通知
              所以，边缘触发需要一次性的把缓冲区的数据读完为止，也就是一直读，直到读到EGAIN为止，EGAIN说明缓冲区已经空了，因为这一点，边缘触发需要设置文件句柄为非阻塞   
            
            + 3.2 epoll的函数调用流程
            
              + 当调用epoll_wait函数的时候，系统会创建一个epoll对象，每个对象有一个evenpoll类型的结构体与之对应，结构体成员结构如下。
                 + rbn,代表将要通过epoll_ctl向epll对象中添加的事件。这些事件都是挂载在红黑树中。
                 + rdlist，里面存放的是将要发生的事件
              + 文件的fd状态发生改变，就会触发fd上的回调函数
              + 回调函数将相应的fd加入到rdlist，导致rdlist不空，进程被唤醒，epoll_wait继续执行。
              + 有一个事件转移函数——ep_events_transfer，它会将rdlist的数据拷贝到txlist上，并将rdlist的数据清空。
              + ep_send_events函数，它扫描txlist的每个数据，调用关联fd对应的poll方法去取fd中较新的事件，将取得的事件和对应的fd发送到用户空间。如果fd是LT模式的话，会被txlist的该数据重新放回rdlist，等待下一次继续触发调用。   
              
            + epoll的优点
            
                + 没有最大并发连接的限制
                + 只有活跃可用的fd才会调用callback函数
                + 内存拷贝是利用mmap()文件映射内存的方式加速与内核空间的消息传递，减少复制开销。（内核与用户空间共享一块内存）
                + 只有存在大量的空闲连接和不活跃的连接的时候，使用epoll的效率才会比select/poll高    