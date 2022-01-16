## 本章目标

- read、write与recv、send
- readline 实现
- 用 readline 实现回射客户/服务器
- getsockname、getpeername
- gethostname、gethostbyname、gethostbyaddr

### read 和 recv：

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t read(int fd, void *buf, size_t count);
```

1、**read、recv 都能从套接口缓冲区获取数据**

2、`read` 可以用于`任何IO`，而 `recv` 只能用于`套接口IO`，不能用于文件IO和其他IO

3、recv 比 read 多了一个` flags 选项`，通过 flags 选项可以指定接受的行为（MSG_OOB 指定接受`带外数据`，也就是通过紧急指针发送的数据、MSG_PEEK 指定接受缓冲区中的数据，但并不清除缓冲区中的数据）

### readline() 函数

首先封装 recv_peek() 函数，为实现 readline() 函数做铺垫。

#### 实现 recv_peek()

功能：能够从套接口中读取数据，但是套接口缓冲区数据不会被清除

```c
ssize_t recv_peek(int sockfd, void *buf, size_t len) {
	while (1) {
		int ret = recv(sockfd, buf, len, MSG_PEEK);
		if (ret == -1 && errno == EINTR)
			continue;
		return ret;
	}
}
```

MSG_PEEK：窥探读缓存中的数据，此次读操作不会导致这些数据被清除

#### readline() 实现方案

readline() 函数只能用于套接口IO，因为内部使用 recv 函数实现。按行读取（也就是说直到遇见 ‘\n’ 才停止读取）

方案1：每次只读取一个字符，判断它是否有\n，但是这种方法会大大增加系统调用的次数，效率较低

方案2：用静态变量保存接受到的数据来进行缓存，下次读入实际上是从缓存中接收数据，但是这样的函数是`不可重入`的（用到了静态变量的函数是有状态的，这样的函数是不可重入的）

方案3：**“偷窥”缓冲区中的数据，遇到 ’\n’ 就从缓冲区中读走数据，没有遇到说明还没有到这条消息的结尾，也读走缓冲区中的数据并保存，然后继续读取**。

```c
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
```

echosrv4.c

```c
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

void do_service(int conn) {
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
			do_service(conn);
			exit(EXIT_SUCCESS);//做完事之后，子进程要退出，否则它也会去接受客户端的连接
		} else {
			close(conn);//对于父进程，不关心连接套接字，只关心监听套接字
		}
	}

	return 0;
}
```

echocli4.c

```c
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
  if (getpeername(conn, (struct sockaddr*)&peeraddr, &peeraddrlen) < 0)
	  ERR_EXIT("getpeername");

  printf("peer ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
	

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
	
	return 0;
}
```

### getsockname, getpeername

```c
//getsockname() 将套接字 sockfd 绑定的当前地址保存至 addr
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

//getpeername() 将连接到套接字 sockfd 的对等方的地址保存至 addr
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

### gethostname、gethostbyname、gethostbyaddr

```c
//gethostname() 获取本地的主机名称
int gethostname(char *name, size_t len);

//gethostbyname() 根据域名或主机名获取IP地址
struct hostent *gethostbyname(const char *name);

struct hostent *gethostbyaddr(const void *addr, socklen_t len, int type);

struct hostent {
               char  *h_name;            /* official name of host */
               char **h_aliases;         /* alias list */
               int    h_addrtype;        /* host address type */
               int    h_length;          /* length of address */
               char **h_addr_list;       /* list of addresses */
};
           #define h_addr h_addr_list[0] /* for backward compatibility */
```

封装一个函数 getlocalip() 获取本地 IP 地址

gethostname_test.c

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
#include <netdb.h>

#define ERR_EXIT(m)\
	do { \
		perror(m);  \
		exit(EXIT_FAILURE);  \
	} while (0)

//获取本机ip
int getlocalip(char *ip) {
	char host[100] = {0};
//gethostname获取主机名称
	if (gethostname(host, sizeof(host)) < 0)
		return -1;
	
	struct hostent *hp;
//gethostbyname用域名或主机名获取IP地址
	if ((hp = gethostbyname(host)) == NULL)
		return -1;

	strcpy(ip, inet_ntoa(*(struct in_addr*)hp->h_addr_list[0]));
	return 0;
}

int main(void) {
/*
	char host[100] = {0};
        if (gethostname(host, sizeof(host)) < 0)
                ERR_EXIT("gethostname");

        struct hostent *hp;
        if ((hp = gethostbyname(host)) == NULL)
                ERR_EXIT("gethostbyname");

	int i = 0;
	while (hp->h_addr_list[i] != NULL) {
		printf("%s\n", inet_ntoa(*(struct in_addr*)hp->h_addr_list[i]));
		++i;
	}
*/
	char ip[16] = {0};
	getlocalip(ip);
	printf("loacl ip = %s\n", ip);
	
	return 0;
}
```
![image](https://user-images.githubusercontent.com/71170476/149647328-84f91604-20cc-41f2-8c83-bb972ec94f15.png)

