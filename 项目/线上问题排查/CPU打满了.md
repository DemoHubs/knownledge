# CPU打满了



## 步骤

- 使用top指令，看看那个进程占用CPU比较高
  - ![image-20191229160807943](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnp2h2vzj31z40pk45w.jpg)
- 使用top -Hp PID查看该进程内的线程执行情况（左边第一列是线程ID）
  - H表示设置线程模式，也就是展示线程的信息
  - p表示展示某个进程的线程信息
  - ![image-20191229160955623](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnqxvz8hj31z40mojyt.jpg)
- 将这个线程ID转换成二进制
  - 通过语句  `printf "%x\n" TID`
  - ![image-20191229161127817](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnsjaj7tj30ji01wwel.jpg)
- 使用jstack找该进程的对应线程的执行情况
  - 通过语句  `jstack 19991 | grep -C30 4e6a --color`
  -  ![image-20191229161312495](https://tva1.sinaimg.cn/large/006tNbRwgy1gadnuck5b9j31z40rwn7j.jpg)
  - 可以看到，由于等待锁（估计是synchronize的轻量级锁空转），导致线程占用CPU变高











## 可能情况

- 代码里面产生了死循环
- 频繁发生Full GC
  - 一次请求生成了很多小对象，并且这些小对象被数组对象强引用着，随着请求的增多，内存逐渐上升，引发MiniGC，后面内存依旧不足，引发了FullGC。













## 参考

[如何排查CPU占用过高以及常见的几种情况](https://blog.csdn.net/coderpopo/article/details/80332496)