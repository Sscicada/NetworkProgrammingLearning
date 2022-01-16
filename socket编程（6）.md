## 本章目标

口TCP回射客户/服务器
口TCP是个流协议
口僵进程与 SIGCHLD信号

## TCP回射客户/服务器
![image](https://user-images.githubusercontent.com/71170476/149643969-7fd7a226-65c3-4768-bf28-3fc4c91f8dba.png)




## TCP是个流协议

口TCP是基于字节流传输的，只维护发送出去多少，确认了多少，没有维护消息与消息之间的辺界，因而可能导致粘包问题。
口粘包问题解决方法是在应用层维护消息边界。

## 僵进程与 SIGCHLD信号

1、忽略SIGCHLD信号可以避免僵尸进程

2、捕捉SIGCHLD信号





![image](https://user-images.githubusercontent.com/71170476/149643996-49da8f15-bedc-458f-a1cc-a057b3242f08.png)







![image](https://user-images.githubusercontent.com/71170476/149644021-28740ea6-68c2-47df-9cc6-a8eaf97a3cb8.png)








发现


![image](https://user-images.githubusercontent.com/71170476/149643948-dc7f46ea-cdc7-48ca-9721-bfb2657a53e5.png)
