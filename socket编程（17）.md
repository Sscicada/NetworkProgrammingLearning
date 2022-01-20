## 本章目标
- `socketpair`
- `sendmsg` / `recvmsg`
- `UNIX` 域套接字传递描述符字

### socketpair

```C
//创建一个全双工的流管道，成功返回0，失败返回-1
int socketpair(int domain, int type, int protocol, int sv[2]);

- domain：协议家族
- type：套接字类型
- protocol：协议类型
- sv：返回套接字对
```
