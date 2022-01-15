## 本章目标

- TCP客户/服务器模型（C/S模型）
- 回射客户/服务器
- socket、bind、listen、 accept、 connect

![image](https://user-images.githubusercontent.com/71170476/149625719-d3480561-8c50-4c00-876d-e4a273b64a19.png)


一旦调用 listen 函数，套接字就变成了被动套接字，否则默认是主动套接字。

- 主动套接字：通过调用 connect 来发起连接
- 被动套接字：通过调用 accept 来接受连接


口一般来说， `listen函数应该在 socket 和 bind 函数之后调用，在 accept 函数之前调用`。

口对于给定的监听套接口，内核要维护两个队列：
1、已由客户发出并到达服务器，服务器正在等待完成相应的 TCP 三路握手过程
2、已完成连接的队列

![image](https://user-images.githubusercontent.com/71170476/149625783-441fcf98-f3d6-4387-a993-7ab9935cc7fe.png)


三次握手时会发送一些 SYN 分节，内核就会开辟一个未完成连接的队列出来，以维护这样的状态。一旦三次握手成功，就会将条目移动到已完成连接队列中。

accept 调用成功之后，又会将相应的条目从已完成连接队列中移除，以便有更多的客户端能够发起连接。

listen 函数的第二个参数规定了能够并发的连接的套接字数目，等于未完成连接数目加上已完成连接数目。listen 函数的第三个参数代表队列的大小，队列大小指的就是未完成连接队列中的条目加上已完成连接队列中的条目。

> 包含头文件<sys/ socket.h>
> 
> 功能：从已完成连接队列返回第一个连接，如果已完成连接队列为空，则阻塞。
> 
> 原型：int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
> 
> 参数
> 
> sockfd：服务器套接字
> 
> addr：将返回对等方的套接字地址
> 
> addrlen：返回对等方的套接字地址长度
> 
> 返回值：成功返回非负整数，失败返回-1

accept 成功将创建一个新的主动套接字，并返回它的文件描述符，注意这个套接字不处于监听状态

accept 函数的第三个参数 perrlen 一定要初始化，因为它是一个输入输出参数，否则 accept 会失败

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
	
	if (bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");

	//SOMAXCONN表示队列的最大值
	if (listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");

	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	if ((conn = accept(listenfd, (struct sockaddr*)& peeraddr, &peerlen)) < 0)
		ERR_EXIT("accept");

	//conn是主动套接字
	printf("ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));

	char recvbuf[1024];
	while (1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int ret = read(conn, recvbuf, sizeof(recvbuf));
		fputs(recvbuf, stdout);
		write(conn, recvbuf, ret);
	}
	close(conn);
	close(listenfd);

	return 0;
}
```

![image](https://user-images.githubusercontent.com/71170476/149626068-61b2f9b6-64c5-40b3-823a-94e35081579e.png)

客户端发送数据，第一次发送的数据比较长，第二次数据比较短，第二次的数据是覆盖写入的，后面有一个换行符。覆盖使默认获得的数据带有换行符

原因在于客户端没有清空缓冲区

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
	
	if (bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");

	//SOMAXCONN表示队列的最大值
	if (listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");

	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	if ((conn = accept(listenfd, (struct sockaddr*)& peeraddr, &peerlen)) < 0)
		ERR_EXIT("accept");

	//conn是主动套接字
	printf("ip = %s port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));

	char recvbuf[1024];
	while (1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int ret = read(conn, recvbuf, sizeof(recvbuf));
		fputs(recvbuf, stdout);
		write(conn, recvbuf, ret);
	}
	close(conn);
	close(listenfd);

	return 0;
}
```

![image](https://user-images.githubusercontent.com/71170476/149626105-b3dfaedb-38f2-4f50-bd1a-45e039422df2.png)
