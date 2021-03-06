# Linux网络IO模型



## 简介

Linux内核将所有的外部设备都看作一个文件来操作，对于一个文件的读写操作会调用内核提供的系统命令，返回一个file descriptor（fd 文件描述符）。而对于一个socket的读写也会有相应的描述符，称为socket fd（socket描述符）。

描述符就是一个数字，它指向内核中的一个结构体（文件路径、数据区等一些属性）。

根据UNIX网络编程对IO模型的分类，UNIX提供了5种IO模型，下面分别介绍。





## 阻塞IO模型

最常用的IO模型就是阻塞IO模型。默认情况下，所有文件操作都是阻塞的。

我们以套接字接口为例讲解此模型：在进程空间中调用recvfrom，其系统调用直到数据包到达并且被复制到应用进程的缓冲区中 或者 发生错误 才会返回。在此期间会一直等待，进程从调用recvform开始到它返回的整段时间内都是被阻塞的，因此被称为阻塞IO模型。

![image-20200503150211526](https://tva1.sinaimg.cn/large/007S8ZIlgy1gef9vbefzuj30w80i20w4.jpg)





## 非阻塞IO模型

应用进程发起recvform系统调用的时候，如果内核数据缓冲区没有数据的话，则直接返回“EWOULDBLOCK”错误，然后应用进程重复发起recvform系统调用来轮询数据是否准备好了。

![image-20200503150639460](https://tva1.sinaimg.cn/large/007S8ZIlgy1gef9zx9zvnj30zw0lk456.jpg)







## IO多路复用模型

Linux提供select/poll，进程通过将一个或者多个fd传递给select或者poll系统调用，阻塞在select操作上，这样子select/poll就可以帮助我们检测多个fd是否处于就绪状态。如果某个fd就绪了，就可以让应用进程进行IO操作。

select/poll是顺序扫描fd是否准备就绪，而且支持的fd数量有限，因此它的使用受到一些制约。Linux还提供了一个epoll系统调用，epoll使用事件驱动的方式检测fd是否就绪，因此性能更高。

![image-20200503151546534](https://tva1.sinaimg.cn/large/007S8ZIlgy1gefa9ermusj311u0nowkc.jpg)

Java NIO的多路复用器Selector就是基于epoll的多路复用技术实现。

在IO编程的过程中，当需要同时处理多个客户端的接入请求时，可以利用多线程或者IO多路复用技术进行处理。

IO多路复用技术通过将多个IO的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求。

与传统的多线程模型相比，IO多路复用的最大优势就是系统开销小，系统不需要创建额外的线程，节省了系统资源。

IO多路复用的主要场景如下：

- 服务器需要同时处理多个处于监听状态 或者 连接状态的套接字。
- 服务器需要同时处理多种网络协议的套接字。



目前支持IO多路复用的系统调用有selec、poll、epoll。









## 信号驱动IO模型

应用进程通过系统调用sigaction执行一个信号处理函数（此系统调用立即返回，进程继续工作，这个操作是非阻塞的）。

当数据准备就绪时，就为该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvform来读取数据，并通知主循环函数处理数据。

![image-20200503151813068](https://tva1.sinaimg.cn/large/007S8ZIlgy1gefabyf7k7j31020mgn2w.jpg)







## 异步IO

应用进程告知内核启动某个操作，并让内核在整个操作（包括将数据从内核复制到用户进程缓存区）完成后通过应用进程。

这种模型和信号驱动模型的主要区别是：

- 信号驱动IO模型由内核通知应用进程何时可以开始一个IO操作；
- 异步IO模型由内核告诉我们IO操作何时已经完成了。

![image-20200503152538027](https://tva1.sinaimg.cn/large/007S8ZIlgy1gefajo3abdj30wo0isgpm.jpg)





## 参考

《Netty权威指南》

[IO模型](https://blog.csdn.net/u011521382/article/details/81040893)

