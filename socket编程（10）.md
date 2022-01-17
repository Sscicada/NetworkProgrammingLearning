## 本章目标 
- close 与 shutdown 区别
- 进一步改进回射客户程序

### close 与 shutdown 的区别

- close：终止了数据传送的两个方向。（仅仅意味着调用 close 的自身不能通过该套接口从全双工管道中写入和读取数据，并不意味着对方不能写入数据）
close 函数会对套接字引用计数减1，一旦发现套接字引用计数到 0，就会对套接字进行释放，并且会关闭 TCP 两个方向的数据流。

- shutdown：可以有选择的终止某个方向的数据传送或者终止数据传送的两个方向。
- shutdown how = 1 就可以保证对等方接收到一个 `EOF` 字符，而不管其他进程是否已经打开了套接字，而 close 不能保证，直到套接字引用计数减为 0 时才发送。也就是说直到所有的进程都关闭了套接字。
（套接字引用计数：因为套接字可以被多个进程共享，你可以理解为我们给每个套接字都设置了一个积分，如果我们通过 fork 的方式产生子进程，套接字就会积分 +1，如果我们调用一次 close 函数，套接字积分就会 -1）

**补充**：

1、`close` 会关闭连接，并*释放所有连接对应的资源*，而 `shutdown `并*不会释放*掉套接字和所有的资源。

2、`close` 存在引用计数的概念，调用 `close` 之后，引用计数减 1 并且本进程不可使用该套接字，但是如果引用计数不为 0 ，那么其他进程仍然可用该套接字；`shutdown` 会直接使该套接字不可用，别的进程也无法使用。

3、`close` 的引用计数导致不一定会发出 `FIN` 结束报文，因为可能还有其他进程在使用套接字，而 **shutdown 总是会发出 FIN 结束报文**，这在我们打算关闭连接通知对端的时候，是非常重要的。

4、shutdown 函数的 howto 选项，有三个主要选项：
- SHUT_RD(0)：关闭连接的“读”这个方向，对该套接字进行读操作直接返回 EOF。从数据角度来看，套接字上接收缓冲区已有的数据将被丢弃，如果再有新的数据流到达，会对数据进行 ACK，然后悄悄地丢弃。也就是说，对端还是会接收到 ACK，在这种情况下根本不知道数据已经被丢弃了。
-  SHUT_WR(1)：**关闭连接的“写”这个方向，这就是常被称为”半关闭“的连接**。此时，不管套接字引用计数的值是多少，都会直接关闭连接的写方向。套接字上发送缓冲区已有的数据将被立即发送出去，并发送一个 FIN 报文给对端。应用程序如果对该套接字进行写操作会报错。
- SHUT_RDWR(2)：相当于 SHUT_RD 和 SHUT_WR 操作各一次，关闭套接字的读和写两个方向


改进后的客户端程序：使用了 shutdown 函数，还有 stdineof ：判断标准输入是否关闭

客户端关闭，那么使用 shutdown 函数只关闭写入，还能接受服务端发来的数据，并发送了一个 FIN 。

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
			//如果从标准输入获得EOF，只关闭写入
			if (fgets(sendbuf, sizeof(sendbuf), stdin) == NULL) {
				shutdown(sock, SHUT_WR);
			} else {
				writen(sock, sendbuf, strlen(sendbuf));
                        	memset(sendbuf, 0, sizeof(sendbuf));
			}
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
}
```

`
