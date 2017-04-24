# 透视I/O多路复用

我写的不是`select`这些函数的教学，需要了解的请自行Google或者去`man`，这些是帮助我理解函数的封装之下的道理。

## 需要回答的问题
1. I/O准备好了指什么？什么叫I/O已经可读/写？

2. 内核如何开始监视？监视的模式如何？

3. 内核以何种机制通知进程？

4. 内核/用户态搬运数据的效率？

5. 阻塞非阻塞？


## 什么是I/O多路复用

I/O多路复用（multiplexing）是一种单进程，单线程监视多个文件描述符（file descriptor）的软件机制。和多进程/多线程的服务方式不同，I/O多路复用将仅有的**单进程阻塞起来**，有一个关注的描述符集合，**内核**在检测到集合中有**准备好读写**的描述符后唤醒进程，进行I/O操作。

现在首先回答第一个问题——怎么叫做**I/O准备好了**？

Linux文档中是这么定义的：
>  A file descriptor is considered ready if it is possible to perform a corresponding I/O operation (e.g., read(2) without blocking, or a sufficiently small write(2)).

此外，`read`和`fread`是默认的“blocking calls”，也就是说当没有可读的数据时进程将转为blocked阻塞状态直到有数据读。

在网络编程中，一个socket（Unix里，socket无差别地是文件描述符）可读的条件是：
> 1. socket内核接收缓存区中的字节数大于或等于其低水位标记SO_RCVLOWAT。此时可以无阻塞地读该socket，并且读操作返回的字节数大于0。
> 2. socket通信对方关闭连接。此时对该socket读操作将返回0。
> 3. 监听socket上有新的连接请求。
> 4. socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。

可写的条件是：
> 1. socket内核发送缓冲区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT。此时我们可以无阻塞写该socket，并且写操作返回的字节数大于0。
> 2. socket写操作被关闭。对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号。
> 3. socket使用非阻塞connect连接成功或者失败（超时）之后。
> 4. socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。

SO_RCVLOWAT和SO_SNDLOWAT是每个socket的两个指标，标识了缓冲超过某个值**内核**才通知进程socket可读。

列举这些诸条的用意是认识到，系统内核有明确的方式和指标对I/O的状态做判断，发现I/O准备好后及时通知进程。

我们已经对**进程需要何种I/O**有了准确的答案，现在回到定义，进程和内核之间的协作如何开展开？这里便引出了2、3两个问题。

## I/O多路复用的实现

### 1. `select/poll`的实现

`select`在调用时调用每一个要监视的fd的poll回调（不是那个系统调用），需要知道，每一个fd有一个队列用来存放所有监视它的进程，这个回调的作用就是给fd的这个队列，叫做`struct wait_queue_head_t`，加入新的监视者。这是`select`的准备工作，到此为止，如果路上遇到了可用的fd那么直接返回，否则进程就可以安心睡会了。
此时，如果受监视的fd上有感兴趣的事情发生（比如字符写入），fd会遍历它的`struct wait_queue_head_t`，唤醒这些监视它的进程——仅仅是唤醒。接下来的事情是进程来做，它重复之前的迭代查询，遍历它的参数集子，确定**所有** ready的fd，收集到返回值里面去，**进程并不知道是谁唤醒的它**，需要再检查的。那么问题3也就得到回答——内核简单唤醒进程，再丢给进程去挨个查询就好。

看着熟悉啊，这不就是visitor模式么？

`poll`在`select`的基础上取消了最大fd数量的限制。其实，在内核文件linux/posix_types.h中有声明曰`#define __FD_SETSIZE    1024`，当然，可以通过重新编译内核手工突破这个限制。

### 2. `epoll`的实现

问题4是一个涉及效率的讨论。`select/poll`机制中，当监视的fd数量巨大时，`select`维护的是一个超大的数据结构，伴随fd数量的增加，成本达到了O(N)级别，此外每次调用时向内核传参数、传出内核消息需要内存拷贝又是一笔开销。`epoll`就是一个解决此问题的出现，适用大量fd同时监听。

顾名思义，`epoll_ctl`意指control，执行它是对常驻内核中的数据结构（红黑树）进行增减操作，而不是每次等待时都传进全部fd。`epoll_ctl`还为相应的fd注册回调函数，事件发生，内核将fd插入一张链表当中，返回给用户。

## 为什么I/O多路复用？

**I/O多路复用适用于多个文件描述符同时监听的情况。**下面基于这一情景。

这首先要聊清楚什么是阻塞不阻塞的I/O（问题5）。阻塞I/O在没有输入时会将进程阻塞，直到读到数据才返回，然而非阻塞I/O在这些不好的情况下会返回一个标识错误的码，保证了进程流畅的进行下去。

前面讲过`read`的默认阻塞行为，那么对于一个没有足够输入的fd，read会将进程阻塞在这一个fd上，大写的不好。

考虑把`read`开成非阻塞式，那么会需要一个循环来不断尝试读，也就是所谓busy waiting，大概长这个样子：
```C++
while(true)
{
    non-blocking read fd1;
    non-blocking read fd2;
    ...
}
```
busy waiting坏处就是长时间占据CPU还可能读不到数据。

用`select`解决这个问题。循环地在**几个fd**上阻塞，注意I/O多路复用可以**同时在多个fd上阻塞**，既满足了多路监听，又有阻塞式的低CPU占用。代码长类似这样：
```C++
while(true)
{
    select_result = select(fds);
    for(auto fd : select_result) deal with fd;
}
```

最后，现在是时候看一下讲了这么久的“多路复用”到底是什么了。wiki上是这么解释的：
> In telecommunications and computer networks, multiplexing (sometimes contracted to muxing) is a method by which multiple analog or digital signals are combined into one signal over a shared medium. The aim is to share an expensive resource.

*多路复用*在一个**昂贵**的设备上组合了多路信号，放在I/O多路复用上就是，用一个阻塞式I/O函数监听多个感兴趣的fd。

此外需要注意，I/O多路复用获得ready的fd并不代表接下来的一切读写都不会阻塞（比如读一个超过SO_RCVLOWAT=100的socket，要从大小为1000的buffer发2000的数据，自信的调阻塞I/O的话就死在这了——当然从`select`的角度讲大可以阻塞I/O直接上），所以该和非阻塞合作的还是要设为非阻塞！
