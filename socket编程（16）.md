## 本章目标
- `UNIX` 域协议特点
- `UNIX` 域地址结构
- `UNIX` 域字节流回射客户/服务
- `UNIX` 域套接字编程注意点

### UNIX 域协议特点
- `UNIX` 域套接字与 `TCP` 套接字相比较，在同一台主机的传输速度前者是后者的两倍。
- `UNIX` 域套接字可以在同一台主机上各进程之间传递描述符。
- `UNIX` 域套接字与传统套接字的区别是用路径名来表示协议族的描述。

### UNIX 域地址结构
```C
struct sockaddr_un {
  		sa_family_t sun_family;               /* AF_UNIX */
		char        sun_path[108];            /* Pathname */
	};
```

### UNIX 域字节流回射客户/服务
echosrv.c
```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/un.h>
#include <string.h>

#define ERR_EXIT(m)\
        do { \
                perror(m);  \
                exit(EXIT_FAILURE);  \
        } while (0)

void echo_srv(int conn) {
	char recvbuf[1024];
	while (1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int n = read(conn, recvbuf, sizeof(recvbuf));
		if (n == -1) {
			if (n == EINTR)
				continue;
			ERR_EXIT("read");
		} else if (n == 0) {
			printf("client close\n");
			break;
		}

		fputs(recvbuf, stdout);
		write(conn, recvbuf, strlen(recvbuf));
	}
	close(conn);
}

int main(void) {
	int listenfd;
	if ((listenfd = socket(PF_UNIX, SOCK_STREAM, 0)) < 0)
		ERR_EXIT("socket");
	
  //unlink()函数会删除所给参数指定的文件
	unlink("/tmp/test_socket");
	struct sockaddr_un servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sun_family = AF_UNIX;
	strcpy(servaddr.sun_path, "/tmp/test_socket");

	if (bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");

	if (listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	int conn;
	pid_t pid;
	while (1) {
		conn = accept(listenfd, NULL, NULL);
		if (conn == -1) {
			if (conn == EINTR)
				continue;
			ERR_EXIT("accept");
		}

		pid = fork();
		if (pid == -1)
			ERR_EXIT("fork");
		else if (pid == 0) {
			echo_srv(conn);
			exit(EXIT_SUCCESS);
		}

		close(conn);
	}
	return 0;
}
```
echocli.c
```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/un.h>
#include <string.h>

#define ERR_EXIT(m)\
        do { \
                perror(m);  \
                exit(EXIT_FAILURE);  \
        } while (0)

void echo_cli(int sock) {
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		write(sock, sendbuf, strlen(sendbuf));
		read(sock, recvbuf, sizeof(recvbuf));
		fputs(recvbuf, stdout);
		memset(sendbuf, 0, sizeof(sendbuf));
		memset(recvbuf, 0, sizeof(recvbuf));
	}
	close(sock);
}

int main(void) {
	int sock;
	if ((sock = socket(PF_UNIX, SOCK_STREAM, 0)) < 0)
		ERR_EXIT("socket");

	struct sockaddr_un servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sun_family = AF_UNIX;
	strcpy(servaddr.sun_path, "/tmp/test_socket");

	if (connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");

	echo_cli(sock);
	return 0;
}
```

### UNIX 域套接字编程注意点
- `bind` 成功将会创建一个文件，权限为 0777 & umask
- `sun path` 最好用一个绝对路径
- `UNIX` 域协议支持流式套接口与报式套接口
- **`UNIX` 域流式套接字 `connect` 发现监听队列满时，会立刻返回一个 `ECONNREFUSED`** ，这和 `TCP` 不同。**对于 `TCP` 来说，客户端调用 `connect` 连接时，如果服务端监听队列满，会忽略到来的 `SYN` ，这导致客户端重传 `SYN` **
