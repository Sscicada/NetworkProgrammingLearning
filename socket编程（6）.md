## 本章目标

- TCP回射客户/服务器

- TCP是个流协议

- 僵进程与 SIGCHLD信号

## TCP回射客户/服务器
![image](https://user-images.githubusercontent.com/71170476/149643969-7fd7a226-65c3-4768-bf28-3fc4c91f8dba.png)


## TCP是个流协议

口TCP是基于字节流传输的，只维护发送出去多少，确认了多少，没有维护消息与消息之间的辺界，因而可能导致粘包问题。

口粘包问题解决方法是在应用层维护消息边界。


## 僵进程与 SIGCHLD信号

方案1：忽略SIGCHLD信号可以避免僵尸进程

方案2：捕捉SIGCHLD信号，进行处理

方案1：
```C
int main(void) {
  signal(SIGCHLD, SIG_IGN);
  ...  
}
```

方案2：
```C
void handle_sigchld(int sig) {
	wait(NULL);
}

int main(void) {
	signal(SIGCHLD, handle_sigchld);
  ...
}
```

wait() 存在的问题：如果有多个子进程同时退出， wait 不能够等待所有子进程的退出，因为它在第一个子进程退出时就返回了，需要用到 waitpid()


在客户端程序模拟 5 个连接

![image](https://user-images.githubusercontent.com/71170476/149643996-49da8f15-bedc-458f-a1cc-a057b3242f08.png)

客户端断开连接

![image](https://user-images.githubusercontent.com/71170476/149644021-28740ea6-68c2-47df-9cc6-a8eaf97a3cb8.png)

五个子进程需要退出，说明有 5 个信号发送给父进程，父进程处于 handle_sigchld 的过程，在处理一个信号的过程中如果有其他信号到达，那么其他信号会丢失，意味着只会排队一个信号，最终只会处理两个子进程，出现 3 个僵尸进程。（这块听得不是太懂？）

```C
void handle_sigchld(int sig) {
	//wait(NULL);
	waitpid(-1, NULL, WNOHANG);
}
```

发现有 3 个僵尸进程

![image](https://user-images.githubusercontent.com/71170476/149643948-dc7f46ea-cdc7-48ca-9721-bfb2657a53e5.png)

```C
void handle_sigchld(int sig) {
	//wait(NULL);
	while (waitpid(-1, NULL, WNOHANG) > 0)
		;
}
```

通过服务端的修改，完美解决僵尸进程

![image](https://user-images.githubusercontent.com/71170476/149644544-cc66f1dc-4e8c-4762-a573-d27144631a50.png)


本章节最终程序：

服务端：主要就是信号处理函数 handle_sigchld() 和封装了 echo_srv()

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

void echo_srv(int conn) {
	char recvbuf[1024];
        while (1) {
                memset(recvbuf, 0, sizeof(recvbuf));
                int ret = readline(conn, recvbuf, 1024);
		if (ret == -1) {
			ERR_EXIT("readline");
		} else if (ret == 0) {//客户端一旦关闭，跳出循环
			printf("client close\n");
                        break;
		}

                fputs(recvbuf, stdout);
                writen(conn, recvbuf, strlen(recvbuf));//回射
        }

}

void handle_sigchld(int sig) {
	//wait(NULL);
	while (waitpid(-1, NULL, WNOHANG) > 0)
		;
}

int main(void) {
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
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	pid_t pid;

	while (1) {
		if ((conn = accept(listenfd, (struct sockaddr*)& peeraddr, &peerlen)) < 0)
                ERR_EXIT("accept");

       		//conn是主动套接字
        	printf("ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
		
		struct sockaddr_in peeraddr;
	        socklen_t peeraddrlen = sizeof(peeraddr);
        	//使用getpeername获取sock绑定的对方的地址和端口
        	if (getpeername(conn, (struct sockaddr*)&peeraddr, &peeraddrlen) < 0)
                	ERR_EXIT("getpeername");

        	printf("client ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));

		pid = fork();
		if (pid == -1)
			ERR_EXIT("fork");
		else if (pid == 0) {//对于子进程，不关心监听套接字，只关心连接套接字
			close(listenfd);
			echo_srv(conn);
			exit(EXIT_SUCCESS);//做完事之后，子进程要退出，否则它也会去接受客户端的连接
		} else {
			close(conn);//对于父进程，不关心连接套接字，只关心监听套接字
		}
	}

	return 0;
}
```

客户端：模拟 5 个并发连接，封装了 echo_cli()

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
	char sendbuf[1024] = {0};
        char recvbuf[1024] = {0};

        //从标准输入中获取数据，并存入sendbuf
        while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
                writen(sock, sendbuf, strlen(sendbuf));//将sendbuf发送给服务器

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
                memset(sendbuf, 0, sizeof(sendbuf));
                memset(recvbuf, 0, sizeof(recvbuf));
        }
        close(sock);

}

int main(void) {
	int sock[5];
	int i;
	for (i = 0; i < 5; ++i) {
		if ((sock[i] = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
              		ERR_EXIT("socket");

        	//servaddr存储地址端口协议家族信息
        	struct sockaddr_in servaddr;
        	memset(&servaddr, 0, sizeof(servaddr));
        	servaddr.sin_family = AF_INET;
        	servaddr.sin_port = htons(5188);
        	//显示地指定服务器的地址
        	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
		
        	if (connect(sock[i], (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
                	ERR_EXIT("connect");
		
        	struct sockaddr_in localaddr;
        	socklen_t addrlen = sizeof(localaddr);
        	//使用getsockname获取sock当前绑定的地址和端口
        	if (getsockname(sock[i], (struct sockaddr*)&localaddr, &addrlen) < 0)
                	ERR_EXIT("getsockname");
		
        	printf("ip = %s port = %d\n", inet_ntoa(localaddr.sin_addr), ntohs(localaddr.sin_port));

        	struct sockaddr_in peeraddr;
        	socklen_t peeraddrlen = sizeof(peeraddr);
        	//使用getpeername获取sock绑定的对方的地址和端口
        	if (getpeername(sock[i], (struct sockaddr*)&peeraddr, &peeraddrlen) < 0)
                	ERR_EXIT("getpeername");
		
        	printf("peer ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
	}

	echo_cli(sock[0]);
	return 0;
}
```


**总结：避免僵尸进程的出现推荐使用捕捉 SIGCHLD 信号的方式来实现。**
