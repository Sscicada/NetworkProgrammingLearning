## 本章目标
- select 限制
- poll

### select 限制
用 select实现的并发服务器，能达到的并发数，受两方面限制：
- 一个进程能打开的最大文件描述符限制。这可以通过调整内核参数来改变

![image](https://user-images.githubusercontent.com/71170476/149728403-a326480d-cd90-436c-8db3-7f3a0c0dfecd.png)

- select 中的 fd_set 集合容量的限制（FD_SETSIZE）。这需要重新编译内核

### poll

```C
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
  int   fd;         /* file descriptor */
  short events;     /* requested events */
  short revents;    /* returned events */
};
```
