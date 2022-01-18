## 本章目标
- 流协议与粘包
- 粘包产生的原因
- 粘包处理方案
- readn writen
- 回射客户/服务器

`TCP` 是基于字节流的传输服务，传输的数据是没有边界的

`UDP` 是基于消息的传输服务，传输的是数据报，具有边界

基于字节流的协议会产生`粘包`问题

### 产生粘包的原因：

1. 应用进程缓冲区中一条消息的大小超过套接口发送缓冲区大小

2. `MSS` 大小限制

3. `MTU` 大小限制

4. 流量控制、拥塞控制

![image](https://user-images.githubusercontent.com/71170476/149870298-6aa8bf94-487a-44ad-8e08-64526c06a015.png)

***解决粘包问题的本质是在应用层维护消息与消息的边界***

### 粘包处理方案：

方案1：接收和发送`定长包`（如 `FTP` 的包尾加 `\r\n`），包头中存有包体的长度。采用封装 `readn` 、 `writen` 函数的做法，读写定长数据。

echosrv.c

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
			if (errno == EINTR)	//被信号中断的情况下，不认为出错，继续读取数据
				continue;
			return -1;
		} else if (nread == 0) {     	//说明对方关闭了
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

void do_service(int conn) {
	char recvbuf[1024];
        while (1) {
                memset(recvbuf, 0, sizeof(recvbuf));
                int ret = readn(conn, recvbuf, sizeof(recvbuf));
		//客户端一旦关闭，跳出循环
		if (ret == 0) {
			printf("client close\n");
			break;
		} else if (ret == -1) {
			ERR_EXIT("read");
		}
                fputs(recvbuf, stdout);
                writen(conn, recvbuf, ret);
        }

}

int main(void) {
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

		pid = fork();
		if (pid == -1)
			ERR_EXIT("fork");
		else if (pid == 0) {//对于子进程，不关心监听套接字，只关心连接套接字
			close(listenfd);
			do_service(conn);
			exit(EXIT_SUCCESS);//做完事之后，子进程要退出，否则它也会去接受客户端的连接
		} else {
			close(conn);//对于父进程，不关心连接套接字，只关心监听套接字
		}
	}

	return 0;
}
```

echocli.c
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

	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	//从标准输入中获取数据，并存入sendbuf
	while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		writen(sock, sendbuf, sizeof(sendbuf));//将sendbuf发送给服务器
		readn(sock, recvbuf, sizeof(recvbuf));//从服务器读取数据并存入recvbuf

		fputs(recvbuf, stdout);//打印服务器传来的数据
		
		//清空缓冲区
		memset(&sendbuf, 0, sizeof(sendbuf));
		memset(&recvbuf, 0, sizeof(recvbuf));
	}
	close(sock);
	
	return 0;
}
```

缺陷：一些消息可能远远小于规定的包体长度，可能会导致网络流量的浪费。

方案2：设计一个头部协议，即自定义一个结构体，代表包，里面存有定长包头与不定长包体，包头存储数据的长度，包体存储数据。先接受定长包头，根据长度接受数据。

服务端
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

struct packet {
	int len;
	char buf[1024];
};

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

void do_service(int conn) {
	struct packet recvbuf;
	int n;
        while (1) {
                memset(&recvbuf, 0, sizeof(recvbuf));
                int ret = readn(conn, &recvbuf.len, 4);//先接受包长并保存至recvbuf.len
		if (ret == -1) {
			ERR_EXIT("readn");
		} else if (ret < 4) {//客户端一旦关闭，跳出循环
			printf("client close\n");
                        break;
		}
		n = ntohl(recvbuf.len);//成功接收到包长后再接受包体内容
		ret = readn(conn, recvbuf.buf, n);
		if (ret == -1) {
                        ERR_EXIT("readn");
                } else if (ret < n) {//客户端一旦关闭，跳出循环
                        printf("client close\n");
                        break;
                }

                fputs(recvbuf.buf, stdout);
                writen(conn, &recvbuf, 4 + n);
        }

}

int main(void) {
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

		pid = fork();
		if (pid == -1)
			ERR_EXIT("fork");
		else if (pid == 0) {//对于子进程，不关心监听套接字，只关心连接套接字
			close(listenfd);
			do_service(conn);
			exit(EXIT_SUCCESS);//做完事之后，子进程要退出，否则它也会去接受客户端的连接
		} else {
			close(conn);//对于父进程，不关心连接套接字，只关心监听套接字
		}
	}

	return 0;
}
```
客户端
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

struct packet {
	int len;
	char buf[1024];
};

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


	struct packet sendbuf;
	struct packet recvbuf;
	memset(&sendbuf, 0, sizeof(sendbuf));
	memset(&recvbuf, 0, sizeof(recvbuf));
	int n;
	//从标准输入中获取数据，并存入sendbuf
	while (fgets(sendbuf.buf, sizeof(sendbuf.buf), stdin) != NULL) {
		n = strlen(sendbuf.buf);
		sendbuf.len = htonl(n);	//发送前写入包中实际内容的长度，统一成网络字节序
		writen(sock, &sendbuf, 4 + n);//将sendbuf发送给服务器
		
		//接受服务器数据
		memset(&recvbuf, 0, sizeof(recvbuf));
                int ret = readn(sock, &recvbuf.len, 4);//先接受包长并保存至recvbuf.len
                if (ret == -1) {
                        ERR_EXIT("readn");
                } else if (ret < 4) {//对方一旦关闭，跳出循环
                        printf("peer close\n");
                        break;
                }
                n = ntohl(recvbuf.len);//成功接收到包长后再接受包体内容
                ret = readn(sock, recvbuf.buf, n);
                if (ret == -1) {
                        ERR_EXIT("readn");
                } else if (ret < n) {//对方一旦关闭，跳出循环
                        printf("peer close\n");
                        break;
                }
                fputs(recvbuf.buf, stdout);
		
		//清空缓冲区
		memset(&sendbuf, 0, sizeof(sendbuf));
		memset(&recvbuf, 0, sizeof(recvbuf));
	}
	close(sock);
	
	return 0;
}
```

