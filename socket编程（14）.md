## 本章目标
- UDP 特点
- UDP 客户/服务基本模型
- UDP 回射客户/服务器
- UDP 注意点

### UDP 特点

1. 无连接
2. 基于消息的数据传输服务
3. 不可靠
4. 一般情況下 UDP 更加高效

### UDP 客户/服务基本模型

![image](https://user-images.githubusercontent.com/71170476/149781488-0e3ed257-d97a-4e9b-9d20-db759b9bd43d.png)

### UDP 回射客户/服务器

![image](https://user-images.githubusercontent.com/71170476/149781712-606ab2a1-9f5a-4fd2-989e-3151499d5f3b.png)

用于 UDP 数据报读写的系统调用是：
```C
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
```
udpsrv.c

```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

#define ERR_EXIT(m)\
        do { \
                perror(m);  \
                exit(EXIT_FAILURE);  \
        } while (0)

void echo_srv(int sock) {
	char recvbuf[1024];
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int n;

	while (1) {
		peerlen = sizeof(peeraddr);
		memset(recvbuf, 0, sizeof(recvbuf));
		n = recvfrom(sock, recvbuf, sizeof(recvbuf), 0, (struct sockaddr*)&peeraddr, &peerlen);
		if (n == -1) {
			if (errno == EINTR)
				continue;
			ERR_EXIT("recvfrom");
		} else if (n > 0) {
			fputs(recvbuf, stdout);
			sendto(sock, recvbuf, n, 0, (struct sockaddr*)&peeraddr, peerlen);
		}
	}
	close(sock);
}

int main(void) {
	int sock;
	if ((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0) {
		ERR_EXIT("socket");
	}
	
	struct sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

	if (bind(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");

	echo_srv(sock);
	return 0;
}
```

udpcli.c

```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

#define ERR_EXIT(m)\
        do { \
                perror(m);  \
                exit(EXIT_FAILURE);  \
        } while (0)

void echo_cli(int sock) {
	struct sockaddr_in servaddr;
        memset(&servaddr, 0, sizeof(servaddr));
        servaddr.sin_family = AF_INET;
        servaddr.sin_port = htons(5188);
        servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		sendto(sock, sendbuf, strlen(sendbuf), 0, (struct sockaddr*)&servaddr, sizeof(servaddr));
		recvfrom(sock, recvbuf, sizeof(recvbuf), 0, NULL, NULL);

		fputs(recvbuf, stdout);
		memset(sendbuf, 0, sizeof(sendbuf));
		memset(recvbuf, 0, sizeof(recvbuf));
	}
	close(sock);
}

int main(void) {
	int sock;
	if ((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0) {
		ERR_EXIT("socket");
	}

	echo_cli(sock);
	return 0;
}
```

### UDP 注意点
- UDP 报文可能会丢失、重复
- UDP 报文可能会乱序
- UDP 缺乏流量控制
- UDP 协议数据报文截断（接收方缓冲区长度需要大于发送方数据长度）
- recvfrom 返回 0 ，不代表连接关闭，因为 UDP 是无连接的。
- ICMP 异步错误
- UDP connect
- UDP 外出接口的确定
