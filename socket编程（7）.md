## 本章目标
- TCP 11种状态
- 连接建立三次握手、连接终止四次握手
- TIME_WAIT 与 SO_REUSEADDR
- SIGPIPE

### TCP 11 种状态

![image](https://user-images.githubusercontent.com/71170476/149645755-739f6dc0-edaa-4e27-a90f-e3fa2c3bfe2d.png)

closing 状态比较特殊：双方需要同时关闭

![image](https://user-images.githubusercontent.com/71170476/149645873-06fb5ad9-8954-4874-9d73-4ab96e0dc158.png)


### SIGPIPE

- 往一个已经接收 FIN 的套接口中写是允许的，接收到 FIN 仅仅代表对方不再发送数据。

- 在收到 RST 段之后，如果再调用 write 就会产生 SIGPIPE 信号，对于这个信号的处理我们通常忽略：
```C
signal(SIGPIPE, SIG_IGN);
```

服务端关闭之后，客户端还可以向套接口中写数据，这会使服务端发送一个 RST 段，如果客户端再次调用 write ，那么就会有 SIGPIPE 信号

```C
void handle_sigpipe(int sig) {
  printf("recv a sig = %d\n", sig);
}

int main(void) {
  //signal(SIGPIPE, SIG_IGN);
  signal(SIGPIPE, handle_sigpipe);
  ...
}
```
