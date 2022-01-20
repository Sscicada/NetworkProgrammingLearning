## 本章目标
- epoll 使用
- epoll 与 select、poll 区别
- epoll LT/ET 模式


### epoll 使用

```C
//创建一个 epoll 实例，返回值: 若成功返回一个大于 0 的值，表示 epoll 实例；若返回 -1 表示出错
int epoll_create(int size);
int epoll_create1(int flags);

//向 epoll 实例增加或删除监控的事件
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

//epoll_wait() 函数类似之前的 poll 和 select 函数，调用者进程被挂起，在等待内核 I/O 事件的分发
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

 typedef union epoll_data {
  void    *ptr;
  int      fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event {
  uint32_t     events;    /* Epoll events */
  epoll_data_t data;      /* User data variable */
};
```


### epoll 与 select、poll 区别

1. 相比于 `select` 与 `poll`, `epoll` 最大的好处在于它不会随着监听 `fd` 数目的增长而降低效率。
2. 内核中的 `select` 与 `poll` 的实现是采用轮询来处理的，即内核要遍历所有的文件描述符，为了找到发生事件的文件描述符。
3. `epoll` 的实现是基于回调的，如果 `fd` 有期望的事件发生，就通过回调函数将其加入 `epoll` 就绪队列中，也就是说它只关心“活跃”的 `fd`，与 `fd` 数目无关。
4. 内核/用户空间内存拷贝问题，**如何让内核把 fd 消息通知给用户空间呢？** 在这个问题上 `select/poll` 采取了`内存拷贝`方法。而 `epoll` 采用了`共享内存`的方式。
5. `epoll` 不仅会告诉应用程序有 I/O 事件到来，还会告诉应用程序相关的信息，这些信息是应用程序填充的，因此根据这些信息应用程序就能直接定位到事件，而不必遍历整个 `fd` 集合。


### epoll LT/ET 模式
1. LT(Level Trigger)：水平/电平触发

   此模式相当于一个效率较高的 `poll` 。完全靠 `kernel epoll` 驱动，应用程序只需要处理从 `epoll_wait` 返回的 `fds` ，这些 `fds` 我们认为它们处于就绪状态。

2. ET(Edge Trigger)：边沿触发
 - 此模式下，系统仅仅通知应用程序哪些 `fds` 变成了就绪状态，一旦 `fd` 变成就绪状态，`epoll` 将不再关注这个 `fd` 的任何状态信息，(从 `epoll` 队列移除)直到应用程序通过读写操作触发  `EAGAIN` 状态，`epoll` 认为这个 `fd` 又变为空闲状态，那么 `epoll` 又重新关注这个 `fd` 的状态变化(重新加入 `epoll` 队列)
 - 随着 `epoll_wait` 的返回，队列中的 `fds` 是在减少的，所以在大并发的系统中， `EPOLLET` 更有优势。但是对程序员的要求也更高

epol1的 LT和ET模式
`epoll` 默认情況下是 `LT` 模式，**在电平触发模式(`LT`)下，若读数据一次性没有读完，缓冲区中还有可读数据，则 `epoll_wait` 还会再次通知**
**若将 `epoll` 设置为 `ET` 模式，若读数据的时候一次性没有读完，则 `epoll_wait` 不再通知，直到下次有新的数据发来**

`ET` 模式由于只通知一次，所以在读的时候要循环读，直到读完，但是当读完之后 `read` 就会阻塞，所以应该将该文件描述符设为非阻塞摸式(`fcntl` 函数)
`read` 函数在非阻塞模式下读的时侯，若返回 -1 ，且 `errno` 为 `EAGAIN` ，则表示当前资源不可用，也就冲区无数据（缓冲区的数据已经读完了）；或者当 `read` 返回的读到的数据长度小于请求的数据长度时，就可以确定此时缓冲区中已没有数据可读了，也就可以认为此时读事件已处理完成。
