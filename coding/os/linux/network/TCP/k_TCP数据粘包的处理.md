
苏丙榅 C语言与CPP编程 _2021年10月22日 08:56_

## 1. 背锅侠 TCP

在前面介绍套接字通信的时候说到了 TCP 是传输层协议，它是一个面向连接的、安全的、流式传输协议。因为数据的传输是基于流的所以发送端和接收端每次处理的数据的量，处理数据的频率可以不是对等的，可以按照自身需求来进行决策。

TCP 协议是优势非常明显，但是有时也会给我们造成困扰，正所谓：成也萧何败萧何。假设我们有如下需求：

> 客户端和服务器之间要进行基于 TCP 的套接字通信
>
> - 通信过程中客户端会每次会不定期给服务器发送一个不定长度的有特定含义的字符串。
>
> - 通信的服务器端每次都需要接收到客户端这个不定长度的字符串，并对其进行解析。

根据上面的描述，服务器在接收数据的时候有如下几种情况：

- 一次接收到了客户端发送过来的一个完整的数据包

- 一次接收到了客户端发送过来的 N 个数据包，由于每个包的长度不定，无法将各个数据包拆开

- 一次接收到了一个或者 N 个数据包 + 下一个数据包的一部分，还是很悲剧，无法将数据包拆开

- 一次收到了半个数据包，下一次接收数据的时候收到了剩下的一部分 + 下个数据包的一部分，更悲剧，头大了

- 另外，还有一些不可抗拒的因素：比如客户端和服务器端的网速不一样，发送和接收的数据量也会不一致

对于以上描述的现象很多时候我们将其称之为 **TCP的粘包问题，但是这种叫法不太对的，本身 TCP 就是面向连接的流式传输协议，特性如此，我们却说是 TCP 这个协议出了问题，这只能说是使用者的无知。多个数据包粘连到一起无法拆分是我们的需求过于复杂造成的，是程序猿的问题而不是协议的问题，TCP 协议表示这锅它不想背**。

现在问题来了，服务器端如果想保证每次都能接收到客户端发送过来的这个不定长度的数据包，程序猿应该如何解决这个问题呢？下面给大家提供几种解决方案：

1. 使用标准的应用层协议（比如：http、https）来封装要传输的不定长的数据包

1. 在每条数据的尾部添加特殊字符，如果遇到特殊字符，代表当条数据接收完毕了

- 有缺陷：效率低，需要一个字节一个字节接收，接收一个字节判断一次，判断是不是那个特殊字符串

3. 在发送数据块之前，在数据块最前边添加一个固定大小的数据头，这时候数据由两部分组成：数据头 + 数据块

- 数据头：存储当前数据包的总字节数，接收端先接收数据头，然后在根据数据头接收对应大小的字节

- 数据块：当前数据包的内容

## 2. 解决方案

如果使用 TCP 进行套接字通信，如果发送的数据包粘连到一起导致接收端无法解析，我们通常使用添加包头的方式轻松地解决掉这个问题。**关于数据包的包头大小可以根据自己的实际需求进行设定，这里没有啥特殊需求，因此规定包头的固定大小为4个字节，用于存储当前数据块的总字节数**。!\[Image\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.1 发送端

对于发送端来说，数据的发送分为 4 步：

1. 根据待发送的数据长度 N 动态申请一块固定大小的内存：N+4（4 是包头占用的字节数）

1. 将待发送数据的总长度写入申请的内存的前四个字节中，此处需要将其转换为网络字节序（大端）

1. 将待发送的数据拷贝到包头后边的地址空间中，将完整的数据包发送出去（字符串没有字节序问题）

1. 释放申请的堆内存。

由于发送端每次都需要将这个数据包完整的发送出去，因此可以设计一个发送函数，如果当前数据包中的数据没有发送完就让它一直发送，处理代码如下：

```cpp
/*   函数描述: 发送指定的字节数   函数参数:       - fd: 通信的文件描述符(套接字)       - msg: 待发送的原始数据       - size: 待发送的原始数据的总字节数   函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1   */   int writen(int fd, const char* msg, int size)   {       const char* buf = msg;       int count = size;       while (count > 0)       {           int len = send(fd, buf, count, 0);           if (len == -1)           {               close(fd);               return -1;           }           else if (len == 0)           {               continue;           }           buf += len;           count -= len;       }       return size;   }   
```

有了这个功能函数之后就可以发送带有包头的数据块了，具体处理动作如下：

```cpp
`/*   函数描述: 发送带有数据头的数据包   函数参数:       - cfd: 通信的文件描述符(套接字)       - msg: 待发送的原始数据       - len: 待发送的原始数据的总字节数   函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1   */   int sendMsg(int cfd, char* msg, int len)   {      if(msg == NULL || len <= 0 || cfd <=0)      {          return -1;      }
// 申请内存空间: 数据长度 + 包头4字节(存储数据长度)   
char* data = (char*)malloc(len+4);      int bigLen = htonl(len);      memcpy(data, &bigLen, 4);      memcpy(data+4, msg, len);      // 发送数据      int ret = writen(cfd, data, len+4);      // 释放内存      free(data);      return ret;   }
```

> 关于数据的发送最后再次强调：**字符串没有字节序问题，但是数据头不是字符串是整形，因此需要从主机字节序转换为网络字节序再发送**。

### 2.2 接收端

了解了套接字的发送端如何发送数据，接收端的处理步骤也就清晰了，具体过程如下：

1. 首先接收 4 字节数据，并将其从网络字节序转换为主机字节序，这样就得到了即将要接收的数据的总长度

1. 根据得到的长度申请固定大小的堆内存，用于存储待接收的数据

1. 根据得到的数据块长度接收固定数目的数据保存到申请的堆内存中

1. 处理接收的数据

1. 释放存储数据的堆内存

从数据包头解析出要接收的数据长度之后，还需要将这个数据块完整的接收到本地才能进行后续的数据处理，因此需要编写一个接收数据的功能函数，保证能够得到一个完整的数据包数据，处理函数实现如下：

```cpp
/*   函数描述: 接收指定的字节数   函数参数:       - fd: 通信的文件描述符(套接字)       - buf: 存储待接收数据的内存的起始地址       - size: 指定要接收的字节数   函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1   */   int readn(int fd, char* buf, int size)   {       char* pt = buf;       int count = size;       while (count > 0)       {           int len = recv(fd, pt, count, 0);           if (len == -1)           {               return -1;           }           else if (len == 0)           {               return size - count;           }           pt += len;           count -= len;       }       return size;   }   
```

这个函数搞定之后，就可以轻松地接收带包头的数据块了，接收函数实现如下：

```cpp
/*   函数描述: 接收带数据头的数据包   函数参数:       - cfd: 通信的文件描述符(套接字)       - msg: 一级指针的地址，函数内部会给这个指针分配内存，用于存储待接收的数据，这块内存需要使用者释放   函数返回值: 函数调用成功返回接收的字节数, 发送失败返回-1   */   int recvMsg(int cfd, char** msg)   {       // 接收数据       // 1. 读数据头       int len = 0;       readn(cfd, (char*)&len, 4);       len = ntohl(len);       printf("数据块大小: %d\n", len);          // 根据读出的长度分配内存，+1 -> 这个字节存储\0       char *buf = (char*)malloc(len+1);       int ret = readn(cfd, buf, len);       if(ret != len)       {           close(cfd);           free(buf);           return -1;       }       buf[len] = '\0';       *msg = buf;          return ret;   }   
```

这样，在进行套接字通信的时候通过调用封装的 sendMsg() 和 recvMsg() 就可以发送和接收带数据头的数据包了，而且完美地解决了粘包的问题。

> 文章链接：https://subingwen.cn/linux/tcp-data-package/#2-2-%E6%8E%A5%E6%94%B6%E7%AB%AF


---


公众号

Read more

Reads 3099

​

Comment

**留言 3**

- 潇洒......

  2021年10月22日

  Like3

  發送部分的方案效率低下，不建議在用戶態判斷是否發送完，而是推給內核處理，特別是在Win和Lin系統![[害羞]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[害羞]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- LHC

  2021年10月22日

  Like2

  挺实用的方法，tcp通讯包含多种报文时，一定得加包含长度的报文头字段。 问一下，这个可以转到朋友圈不？

  C语言与CPP编程

  Author2021年10月22日

  Like

  欢迎转发

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/ibLeYZx8Co5JKf72TOeLcba56VknmOtKrMWnS3gyv2Z3RPZ6S28sAtAKSyozOHMDzI8LEkz8ic8eH2v4ZysDq6sQ/300?wx_fmt=png&wxfrom=18)

C语言与CPP编程

20810

3

Comment

**留言 3**

- 潇洒......

  2021年10月22日

  Like3

  發送部分的方案效率低下，不建議在用戶態判斷是否發送完，而是推給內核處理，特別是在Win和Lin系統![[害羞]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[害羞]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)

- LHC

  2021年10月22日

  Like2

  挺实用的方法，tcp通讯包含多种报文时，一定得加包含长度的报文头字段。 问一下，这个可以转到朋友圈不？

  C语言与CPP编程

  Author2021年10月22日

  Like

  欢迎转发

已无更多数据
