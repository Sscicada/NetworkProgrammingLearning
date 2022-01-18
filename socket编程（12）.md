## 本章目标
- select 限制
- poll

### select 限制
用 select实现的并发服务器，能达到的并发数，受两方面限制：
- 一个进程能打开的最大文件描述符限制。这可以通过调整内核参数来改变

![image](https://user-images.githubusercontent.com/71170476/149913050-dc69dbba-1cb0-432b-a4a3-6a99f38d3aa0.png)

通过 ulimit -n 命令查看一个进程能打开的最大文件描述符数目：

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
- fds 是一个数组，存储我们感兴趣的套接字
- nfds 是数组中元素的个数
- timeout 是超时时间

pollsrv.c
```C
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <stdio.h>
#include <signal.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
#include <poll.h>

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
			if (errno == EINTR)	//被信号中断的情况下，不认为出错，继续读取数据
				continue;
			return -1;
		} else if (nread == 0) {     	//说明对方关闭了
			int x = count;
			if (count == nleft) printf("count = %d\n", x);
			return count - nleft;
		}
		bufp += nread;
		nleft -= nread;
	}
	if (count == 0) printf("count == 0\n");
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

void handle_sigchld(int sig) {
	//wait(NULL);
	while (waitpid(-1, NULL, WNOHANG) > 0)
		;
}

void handle_sigpipe(int sig) {
	printf("recv a sig = %d\n", sig);
}

int main(void) {
	signal(SIGCHLD, handle_sigpipe);
	signal(SIGCHLD, handle_sigchld);
	//signal(SIGCHLD, SIG_IGN);
	int listenfd;
	//if ((listenfd = socket(PF_INET, SOCK_STREAM, 0)) < 0)
	if ((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
	       ERR_EXIT("socket");

	//servaddr存储地址端口协议家族信息
	struct sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	//servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	//inet_aton("127.0.0.1", &servaddr.sin_addr);
	
	int on = 1;
	if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
		ERR_EXIT("setsockopt");
	
	if (bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");

	if (listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int conn;
	int i, maxi = 0;		//maxi记录client数组中最大不空闲位置，维护这个变量的目的是
					//提高一点效率，每次不用遍历到client数组结束
	struct pollfd client[2048];	//保存所有已连接的客户端套接口

	for (i = 0; i < 2048; ++i) {
		client[i].fd = -1;
	}

	int nready;			//存储poll结果
	client[0].fd = listenfd;	//对监听套接口的可读事件感兴趣
	client[0].events = POLLIN;
	while (1) {
		nready = poll(client, maxi + 1, -1);
		if (nready == -1) {
			if (errno == EINTR)
				continue;
			ERR_EXIT("select");
		} else if (nready == 0)
			continue;

		//监听套接口发生可读事件，意味着三次握手已经完成了，对方connect成功
		//本地已完成连接队列不为空，accept不再阻塞，尝试与对方建立连接
		if (client[0].revents & POLLIN) {
			peerlen = sizeof(peeraddr);
			conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen);
			if (conn == -1)
				ERR_EXIT("accept");
			
			//建立连接成功，寻找空闲位置，以存储连接套接口
			for (i = 0; i < 2048; ++i) {
				if (client[i].fd < 0) {
					client[i].fd = conn;
					//更新最大不空闲位置
					if (i > maxi)
						maxi = i;
					break;
				}
			}

			//连接数量超过上限，打印错误信息
			if (i == 2048) {
				fprintf(stderr, "too many clients\n");
				exit(EXIT_FAILURE);
			}

			printf("ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
			
			client[i].events = POLLIN;
			//事件处理完就可以直接进入下一轮
			if (--nready <= 0)
				continue;
		}
		
		//处理已连接套接口的事件
		for (i = 1; i <= maxi; ++i) {
			conn = client[i].fd;
			if (conn == -1)
				continue;
			//如果有可读事件需要处理
			if (client[i].revents & POLLIN) {
				char recvbuf[1024] = {0};
				int ret = readline(conn, recvbuf, 1024);
		                if (ret == -1) {
                		        ERR_EXIT("readline");
                		} else if (ret == 0) {//客户端一旦关闭，从client数组中移除
                        		printf("client close\n");
					client[i].fd = -1;
					close(conn);
                		}
				
                		fputs(recvbuf, stdout);
                		writen(conn, recvbuf, strlen(recvbuf));//回射

				if (--nready <= 0)
					break;
			}
		}
	}
	return 0;
}
```
