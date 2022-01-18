## 本章目标

- REUSEADDR
- 处理多客户连接( process-per-connection)
- 点对点聊天程序实现

一个 `TCP` 连接对应于一个四元组，包括服务端 `IP` 、服务端端口、客户端 `IP` 、客户端端口。

回响服务器端程序还存在一些问题

第一个问题：使用服务端并关闭之后，暂时不能重新启动服务端。因为服务端重新启动时，肯定要去绑定地址，但是这时地址是没办法绑定成功的，因为服务端处于 `TIME_WAIT` 状态。

![image](https://user-images.githubusercontent.com/71170476/149871158-b84ee355-8c75-4872-b9ea-8be2685d6062.png)

```c
int on = 1;
if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
      ERR_EXIT("setsockopt");
```

解决：使用 `RESUSEADDR` 选项（这个选项的含义是在 `TIME_WAIT` 状态还没有结束时，允许服务端重启）

口 服务器端尽可能使用 `REUSEADDR`
口 在绑定之前尽可能调用 `setsockopt` 来设置 `REUSEADDR` 套接字选项。
口 使用 `REUSEADDR` 选项可以使得不必等待 `TIME_WAIT` 状态消失就可以重启服务器。

第二个问题：无法处理多个客户端的连接

解决：当父进程 `accept` 成功之后，`fork` 一个子进程来处理通信的细节，父进程继续等待其他客户端的连接，这样就可以处理多个客户端的连接了。

![image](https://user-images.githubusercontent.com/71170476/149871193-4545dcb2-e452-4934-b6c5-faa972d58fb0.png)

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

void do_service(int conn) {
	char recvbuf[1024];
        while (1) {
                memset(recvbuf, 0, sizeof(recvbuf));
                int ret = read(conn, recvbuf, sizeof(recvbuf));
								//客户端一旦关闭，跳出循环
		            if (ret == 0) {
									printf("client close\n");
									break;
								} else if (ret == -1) {
									ERR_EXIT("read");
								}
                fputs(recvbuf, stdout);
                write(conn, recvbuf, ret);
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

![image](https://user-images.githubusercontent.com/71170476/149871272-ac8bff96-3cee-4449-a7db-6d52a06d6a0a.png)


### 实现点对点聊天程序

双方都能互相发送数据，都维护套接字，但是要想实现一方在接收对方发送数据的同时，还能发送数据给对方（即获取键盘上的输入），可以创建进程出来，一个进程处理接收数据，一个进程获取键盘输入。

用信号实现进程间通信

![image](https://user-images.githubusercontent.com/71170476/149871360-f345fca8-eed0-4857-b63a-a4003249e5bd.png)


p2psrv.c
```C
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
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

void handler(int sig) {
	printf("recv a sig = %d\n", sig);
	exit(EXIT_SUCCESS);
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
	if ((conn = accept(listenfd, (struct sockaddr*)& peeraddr, &peerlen)) < 0)
		ERR_EXIT("accept");

	//conn是主动套接字
	printf("ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
	
	pid_t pid;
	pid = fork();
	if (pid == 1)
		ERR_EXIT("fork");
	//处理键盘输入
	else if (pid == 0) {
    signal(SIGUSR1, handler);
		char sendbuf[1024] = {0};
		while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
			write(conn, sendbuf, strlen(sendbuf));
			memset(sendbuf, 0, sizeof(sendbuf));
		}
		printf("child close\n");
        	exit(EXIT_SUCCESS);
	} else {     //处理接受数据
    char recvbuf[1024];
    while (1) {
      memset(recvbuf, 0, sizeof(recvbuf));
      int ret = read(conn, recvbuf, sizeof(recvbuf));
      if (ret == -1)
        ERR_EXIT("read");
      else if (ret == 0) {
        printf("peer close\n");
				break;
      } 
      fputs(recvbuf, stdout);
      write(conn, recvbuf, ret);
    }
		printf("parent close\n");
		kill(pid, SIGUSR1);
		exit(EXIT_SUCCESS);
	}
	close(conn);
	close(listenfd);

	return 0;
}
```

p2pcli.c
```C
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
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

void handler(int sig) {
	printf("recv a sig = %d\n", sig);
	exit(EXIT_SUCCESS);
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


	pid_t pid;
	pid = fork();
	if (pid == -1)
		ERR_EXIT("fork");
	else if (pid == 0) {
		char recvbuf[1024] = {0};
		while (1) {
			memset(recvbuf, 0, sizeof(recvbuf));
			int ret = read(sock, recvbuf, sizeof(recvbuf));
			if (ret == -1)
				ERR_EXIT("read");
			else if (ret == 0) {
				printf("peer close\n");
				break;
			}
			fputs(recvbuf, stdout);
		}
		kill(getppid(), SIGUSR1);
		close(sock);
	} else {
		signal(SIGUSR1, handler);
		char sendbuf[1024] = {0};
    //从标准输入中获取数据，并存入sendbuf
    while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
      write(sock, sendbuf, strlen(sendbuf));//将sendbuf发送给服务器
			
      //清空缓冲区
      memset(&sendbuf, 0, sizeof(sendbuf));
    }
    close(sock);
	}

	return 0;
}
```
