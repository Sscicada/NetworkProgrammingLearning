## 本章目标

- 什么是 socket
- IPV4套接口地址结构
- 网络字节序
- 字节序转换函数
- 地址转换函数
- 套接字类型

口 socket可以看成是用户进程与内核网络协议栈的编程接口。

口 socket不仅可以用于本机的进程间通信，还可以用于网络上不同主机的进程间通信，也就是说可以把socket看成是进程间通信的一种手段。

```c
struct sockaddr_in {
	uint8_t sin_len;    //结构体的长度
	sa_family_t sin_family;   //地址家族,必须设为AF_INET
	in_port_t sin_port;       //端口号
	struct in_addr sin_addr;  //IPV4地址
	char sin_zero[8];         //暂不使用
};
```

![image](https://user-images.githubusercontent.com/71170476/149625249-25adce2e-1c1f-49c7-bf0b-b90f67d5d1eb.png)

![image](https://user-images.githubusercontent.com/71170476/149625330-338b91c1-c757-41dc-8fcb-774929838324.png)

![image](https://user-images.githubusercontent.com/71170476/149625358-97a14831-73c2-4e02-9218-fa9b15b7c659.png)

口字节序

- 大端字节序：高位字节在低地址，低位字节在高地址。这是人类读写数值的方法。

- 小端字节序：低位字节在低地址，高位字节在高地址

x86平台一般为小端字节序，但网络字节序规定为大端字节序。

```c
#include <stdio.h>
#include <arpa/inet.h>

int main(void) {
	unsigned int x = 0x12345678;
	unsigned char *p = (unsigned char*)&x;
	printf("%0x %0x %0x %0x\n", p[0], p[1], p[2], p[3]);
	
	unsigned int y = htonl(x);
	p = (unsigned char*)&y;
	printf("%0x %0x %0x %0x\n", p[0], p[1], p[2], p[3]);
	return 0;
}
```

![image](https://user-images.githubusercontent.com/71170476/149625423-c953b9b4-29a0-44e5-9699-5b2c75250d23.png)

- 流式套接字(SOCK_ STREAM)
提供面向连接的、可靠的数据传输服务，数据无差错，无重复的发送，且按发送顺序接收
- 数据报式套接字(SOCK_ DGRAM)
提供无连接服务。不提供无错保证，数据可能丢失或重复，并且接收顺序混乱
- 原始套接字( SOCK_RAW)
可以跨越传输层，直接对IP层进行数据封装。通过原始套接字，我们可以将应用层的数据直接封装成IP层可以认识的协议格式

![image](https://user-images.githubusercontent.com/71170476/149625497-0f48d62a-e425-47df-a2c4-2c981d74ff99.png)

```c
#include <stdio.h>
#include <arpa/inet.h>

int main(void) {
	unsigned long addr = inet_addr("192.168.0.100");      //将点分十进制的IP地址转换成网络字节序的32位的整数
	printf("addr = %u\n", ntohl(addr));
	
	struct in_addr ipaddr;
	ipaddr.s_addr = addr;
	printf("%s\n", inet_ntoa(ipaddr));		   //将网络字节序的32位的整数转换成点分十进制的IP地址

	return 0;
}
```

![image](https://user-images.githubusercontent.com/71170476/149625518-e8b02cdd-a20c-4a3b-8267-fdc913f53acd.png)
