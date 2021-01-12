# Happen-Before原则
    java内存模型是共享内存的并发模型，线程之间主要通过读-写共享变量来完成隐式通信。
    java中的共享变量是存储在内存中的，多个线程由其工作内存，其工作方式是将共享内存中的变量拿出来放在工作内存，操作完成后，
    再将最新的变量放回共享变量，这时其他的线程就可以获取到最新的共享变量。

   ![](https://outman-1252077993.cos.ap-nanjing.myqcloud.com/stickPicture%20(1).png)


# 什么是Happen-Before
    JMM可以通过happens-before关系向程序员提供跨线程的内存可见性保证（如果A线程的写操作a与B线程的读操作b之间存在happens-before关系，尽管a操作和b操作在不同的线程中执行，但JMM向程序员保证a操作将对b操作可见）。

具体的定义为：

    1）如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
    2）两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。

具体的规则：

    (1)程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。（外部观察是有序的，而在单线程内部可能重排序）
    (2)监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
    (3)volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
    (4)传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
    (5)start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
    (6)Join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
                
```
    public void joinDemo(){
     //....
     Thread t=new Thread(payService);
    t.start();
    //.... 
    //其他业务逻辑处理,不需要确定t线程是否执行完
     insertData();
    //后续的处理，需要依赖t线程的执行结果，可以在这里调用join方法等待t线程执行结束
    t.join();	
    }
```
    (7)程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
    (8)对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。