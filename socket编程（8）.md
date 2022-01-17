## 本章目标
- 五种 I/O 模型
- select
- 用 select 改进回射客户端程序

### 五种 I/O 模型
- 阻塞 I/O
- 非阻塞 I/O
- I/O 复用（select 和 poll）
- 信号驱动 I/O
- 异步 I/O

#### 阻塞 I/O

![image](https://user-images.githubusercontent.com/71170476/149646685-3918db46-8217-42c9-acf7-14724c65d8ad.png)

#### 非阻塞 I/O
（不推荐使用，应用范围很窄）

可以将套接口设置成非阻塞模式。即使没有数据到来， recv 函数也不会阻塞。

这种模型会不断循环判断 recv 返回值，如果为 -1 ，说明没有数据到来，继续循环，直到有数据，这种现象被称为`忙等待`。

```C
fcntl(fd, F_SETFL, flag|O_NONBLOCK);
```

![image](https://user-images.githubusercontent.com/71170476/149646731-eb6ce5b2-2d4e-4b6d-8009-d31da074d4e5.png)

#### I/O 复用

用 select 函数来管理多个文件描述符，一旦其中某个文件描述符有数据到来，select 就返回（阻塞的位置提前到 select）

![image](https://user-images.githubusercontent.com/71170476/149646994-2605f36d-d7a8-4275-b24b-7407b01fd44e.png)

#### 信号驱动 I/O

![image](https://user-images.githubusercontent.com/71170476/149647043-41214ad2-a750-4b69-8a71-56148261ac8d.png)



#### 异步 I/O

![image](https://user-images.githubusercontent.com/71170476/149647476-8fded635-4956-46f6-9b3e-b29c4e55fef2.png)


### select
`用 select 来管理多个 I/O`，一旦其中的一个或多个 I/O 检测到我们感兴趣的事件，select 就返回，返回值为检测到的事件个数。

```C
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
- nfds：读、写、异常集合中的文件描述符的最大值加 1
- readfds：读集合。被放到该集合的文件描述符，意味着我们关心它们的可读事件
- writefds：写集合
- exceptfds：异常集合
- timeout：超时结构体。设置超时时间

*后四个参数都是输入输出参数*

```C
void FD_CLR(int fd, fd_set *set);     //从相应的集合中移除相应的文件描述符
int  FD_ISSET(int fd, fd_set *set);   //判断某文件描述符是否在某集合中
void FD_SET(int fd, fd_set *set);     //添加文件描述符到集合中
void FD_ZERO(fd_set *set);            //清空集合
```

### 用 select 改进回射客户端程序

改进了 echo_cli()：使用了 select ，监听标准输入和套接口，对等方发送的数据可以及时得到处理


```C
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

#define ERR_EXIT(m)\
	do { \
		perror(m);  \
		exit(EXIT_FAILURE);  \
	} while (0)

ssize_t readn(int fd, void *buf, size_t count) {
        size_t nleft = count;//还剩多少字节的数据需要读
        ssize_t nread;
        char *bufp = (char*)buf;

        while (nleft > 0) {
                if ((nread = read(fd, bufp, nleft)) < 0) {//将读到的数据存入bufp中
                        if (errno == EINTR)     //被信号中断的情况下，不认为出错，继续读取数据
                                continue;
                        return -1;
                } else if (nread == 0) {        //说明对方关闭了
                        return count - nleft;
                }
                bufp += nread;
                nleft -= nread;
        }
        return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
        size_t nleft = count;
        ssize_t nwrite;
        char *bufp = (char*)buf;

        while (nleft > 0) {
                if ((nwrite = write(fd, bufp, nleft)) < 0) {
                        if (errno == EINTR)
                                continue;
                        return -1;
                } else if (nwrite == 0) {
                        continue;
                }
                bufp += nwrite;
                nleft -= nwrite;
        }
        return count;
}

ssize_t recv_peek(int sockfd, void *buf, size_t len) {
        while (1) {
                int ret = recv(sockfd, buf, len, MSG_PEEK);
                if (ret == -1 && errno == EINTR)
                        continue;
                return ret;
        }
        return -1;
}

ssize_t readline(int sockfd, void *buf, size_t maxline) {
        int ret, nread;
        char *bufp = buf;
        int nleft = maxline;
        while (1) {
                ret = recv_peek(sockfd, bufp, nleft);   //偷窥数据，即保存缓冲区中的数据至buf，但是不清除缓冲区中的数据
                if (ret < 0)
                        return ret;
                else if (ret == 0)
                        return ret;

                nread = ret;            //读取到的字节数
                int i;
                for (i = 0; i < nread; ++i) {
                        //遇到'\n'，说明消息结束，读走缓冲区中的数据
                        if (bufp[i] == '\n') {
                                ret = readn(sockfd, bufp, i + 1);
                                if (ret != i + 1)
                                        exit(EXIT_FAILURE);
                                return ret;
                        }
                }

                if (nread > nleft)
                        exit(EXIT_FAILURE);
                nleft -= nread;
                ret = readn(sockfd, bufp, nread);
                if (ret != nread)
                        exit(EXIT_FAILURE);

                bufp += nread;
        }
        return -1;
}

void echo_cli(int sock) {
	fd_set rset;
	FD_ZERO(&rset);

	int nready;
	int maxfd;
	int fd_stdin = fileno(stdin);
	if (fd_stdin > sock)
		maxfd = fd_stdin;
	else
		maxfd = sock;

	char sendbuf[1024] = {0};
        char recvbuf[1024] = {0};

	//不断循环检测事件
	while (1) {
		//加入集合
		FD_SET(fd_stdin, &rset);
		FD_SET(sock, &rset);
		nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
		if (nready == -1)
			ERR_EXIT("select");
		else if (nready == 0)	//没有事件
			continue;

		//检测到事件，再判断是否在rset中
		if (FD_ISSET(sock, &rset)) {
			//接受服务器数据
                	int ret = readline(sock, recvbuf, sizeof(recvbuf));
                	if (ret == -1) {
                        	ERR_EXIT("readline");
                	} else if (ret == 0) {//对方一旦关闭，跳出循环
                        	printf("peer close\n");
                        	break;
                	}
                	fputs(recvbuf, stdout);

                	//清空缓冲区
                	memset(recvbuf, 0, sizeof(recvbuf));
		}
		
		if (FD_ISSET(fd_stdin, &rset)) {
			//如果从标准输入获得EOF
			if (fgets(sendbuf, sizeof(sendbuf), stdin) == NULL)
				break;
			writen(sock, sendbuf, strlen(sendbuf));
			memset(sendbuf, 0, sizeof(sendbuf));
		}
	}
	close(sock);
}

int main(void) {
	int sock;
	if ((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
	       ERR_EXIT("socket");

	//servaddr存储地址端口协议家族信息
	struct sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	//显示地指定服务器的地址
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");

	if (connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
	
	struct sockaddr_in localaddr;
	socklen_t addrlen = sizeof(localaddr);
	//使用getsockname获取sock当前绑定的地址和端口
	if (getsockname(sock, (struct sockaddr*)&localaddr, &addrlen) < 0)
		ERR_EXIT("getsockname");
	
	printf("ip = %s port = %d\n", inet_ntoa(localaddr.sin_addr), ntohs(localaddr.sin_port));
	
	struct sockaddr_in peeraddr;
	socklen_t peeraddrlen = sizeof(peeraddr);
	//使用getpeername获取sock绑定的对方的地址和端口
	if (getpeername(sock, (struct sockaddr*)&peeraddr, &peeraddrlen) < 0)
		ERR_EXIT("getpeername");

	printf("peer ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));

	echo_cli(sock);	
	return 0;
```

**如果需要处理多个 I/O ，又想用单进程的方式实现，那么用 select 比较方便**
