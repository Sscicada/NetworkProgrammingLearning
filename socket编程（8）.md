## 本章目标
- 五种 I/O 模型
- select
- 用 select 改进回射客户端程序

### 五种 I/O 模型
- 阻塞 I/O
- 非阻塞 I/O
- I/O 复用（select 和 poll）
- 信号驱动 I/O
- 异步 I/O

#### 阻塞 I/O

![image](https://user-images.githubusercontent.com/71170476/149646685-3918db46-8217-42c9-acf7-14724c65d8ad.png)

#### 非阻塞 I/O
（不推荐使用，应用范围很窄）

可以将套接口设置成非阻塞模式。即使没有数据到来， recv 函数也不会阻塞。

这种模型会不断循环判断 recv 返回值，如果为 -1 ，说明没有数据到来，继续循环，直到有数据，这种现象被称为`忙等待`。

```C
fcntl(fd, F_SETFL, flag|O_NONBLOCK);
```

![image](https://user-images.githubusercontent.com/71170476/149646731-eb6ce5b2-2d4e-4b6d-8009-d31da074d4e5.png)

#### I/O 复用

用 select 函数来管理多个文件描述符，一旦其中某个文件描述符有数据到来，select 就返回（阻塞的位置提前到 select）

![image](https://user-images.githubusercontent.com/71170476/149646994-2605f36d-d7a8-4275-b24b-7407b01fd44e.png)

#### 信号驱动 I/O
（不经常使用）

![image](https://user-images.githubusercontent.com/71170476/149647043-41214ad2-a750-4b69-8a71-56148261ac8d.png)



#### 异步 I/O




