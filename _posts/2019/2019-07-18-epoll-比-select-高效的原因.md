---
layout: post
title: epoll 比 select 高效的原因
---

epoll 比 select 高效的原因还是要从他们的实现上来说.

首先, linux 下文件描述符都可以在其状态就绪 (有新的数据, 可以写入数据等) 时, 执行一个指定的回调函数.
select 和 epoll 的实现都是基于这一机制, 区别就在于如何利用这一回调机制.

#### select

当我们调用 select, 并传入一批文件描述符, 假设当前所有文件描述符都不就绪, select 会在所有这些文件描述符上加上一个回调函数, 
然后将当前进程睡眠. 回调函数的功能很简单, 就是唤醒睡眠的进程. 进程唤醒后继续执行, 会轮询检查监控的文件描述符是否就绪,
同时将文件描述符上对应的回调函数移除.

#### epoll

和 select 不同的地方是, epoll 不需要频繁的在监控的文件描述符上添加移除回调函数.
只有当你调用 epoll_ctl 添加监控的文件描述符时 epoll 才会在对应的文件描述符上添加一个回调函数.
相应的, 调用 epoll_ctl 删除监控的文件描述符时会将回调函数移除. 另外需要说明的是, 调用 epoll_create 的时候,
epoll 会创建一个文件描述符, 当你调用 epoll_wait 的时候, 会在 epoll 创建的文件描述符上插入一个回调函数,
然后将当前进程睡眠.

epoll 插入到监控的文件描述符上的回调函数所做的事情是, 将当前就绪的文件描述符放到 epoll 自己维护的一个红黑树里,
同时执行 epoll 自己创建的文件描述符上的回调函数, 这个回调函数会将睡眠的进程唤醒. 唤醒的进程会继续执行 epoll_wait 函数, 
将之前插入的回调函数移除.

#### 总结

可以发现, epoll 比 select 高效的原因是, a) 不需要频繁的在监控的文件描述符上添加移除回调函数
b) 可以直接获取就绪的文件描述符, 不需要遍历检查每个文件描述符

#### 等待队列

上面的 select 和 epoll 的实现都基于文件描述符在就绪时执行回调函数, 那么这个又是怎么实现的呢?

其实也很简单, 每个文件描述符维护了一个等待队列 (wait_queue_head). 所有的文件描述符都对外暴露一个 file_operations 的结构体 (具体实现是各个 IO 设备的驱动实现的), 其中有一个 poll 方法 (和系统调用 poll 不是同一个东西). 调用这个 poll 方法, 
会在等待队列中插入一个等待队列项 (wait_queue), 等待队列项中就有我们要执行的回调函数.
这样当对应的文件描述符状态改变 (写入, 读取等操作) 时就会遍历等待队列并执行相应的回调函数.

#### 其他

用惯了高层语言, 看 select 和 epoll 的源码的时候, 总是会带入一些已经抽象过的概念, 比如进程.
对于内核代码来说, 进程不过就是内存中的一块数据而已, 所有的进程调度, 都是需要内核代码自己实现.

很久不用 c 了, 对 c 的很多语法都忘了. 比如一开始好奇 epoll 的回调函数里怎么拿到对应的文件描述符的,
后来发现原来是因为 wait_queue 是属于一个结构体的, 回调函数中直接基于偏移获取 wait_queue 所属的结构体,
再获取结构体中的文件描述符, 这个不就是闭包的底层实现嘛!

看源码的时候总是看了后面忘前面, 后来发现还是要把代码图形化, 画 UML 图是个好方法.
所以记忆还是要基于图形化!

> [源码解读Linux等待队列](http://gityuan.com/2018/12/02/linux-wait-queue/)
>
> [源码解读poll/select内核机制](http://gityuan.com/2019/01/05/linux-poll-select/)
>
> [源码解读epoll内核机制](http://gityuan.com/2019/01/06/linux-epoll/)
