#

原创 盼盼编程 盼盼编程

 _2021年12月03日 22:58_

大家好，我是盼盼！

  

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。  
  
相对于select和poll来说，epoll更加灵活，没有描述符限制。  
  
epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。  
  
学一下redis源码中epoll的使用。

```
/*
```

```
/*
```

```
/* State of an event based program 
```

aeApiCreate函数为epoll_event分配空间，epoll_create创建一个epoll句柄，参数size用来告诉内核监听的文件描述符的个数，epoll句柄就是函数的返回值。  
  
该返回值保存在aeEventLoop的apidata里面。

  

```
/*
```

epoll_ctl是事件注册函数，注册要监听的事件类型。state->epfd是epoll_create的返回值，op表示动作，EPOLL_CTL_ADD：注册新的fd到epfd中。  
  
EPOLL_CTL_MOD：修改已经注册的fd的监听事件。  
  
EPOLL_CTL_DEL：从epfd中删除一个fd；第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事。通信每过一个阶段，监听事件的状态也会发生改变，通过epoll_ctl来改变。

  

```
typedef union epoll_data {
```

epoll_wait等待事件的产生参数events用来从内核得到事件的集合，setsize告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数tvp是超时时间。

![](http://mmbiz.qpic.cn/mmbiz_png/bZqlQqGqRvXQmv3GfYP2icpUAh9l4rwA93YDxJhRkvtHKyE6TA91BXQf9pRribGtBh1S3jI1FWiau8xLrVm19MxAQ/300?wx_fmt=png&wxfrom=19)

**盼盼编程**

专注分享编程学习，校招求职和大厂面试！

28篇原创内容

公众号

阅读 404

​

写留言

**留言 2**

- Henry
    
    2021年12月4日
    
    赞
    
    这是剖析的源码，可以写一个hello world 实际跑一下，打印数据看看吗？
    
    盼盼编程
    
    作者2021年12月4日
    
    赞
    
    直接运行redis就行
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/bZqlQqGqRvXQmv3GfYP2icpUAh9l4rwA93YDxJhRkvtHKyE6TA91BXQf9pRribGtBh1S3jI1FWiau8xLrVm19MxAQ/300?wx_fmt=png&wxfrom=18)

盼盼编程

6分享4

2

写留言

**留言 2**

- Henry
    
    2021年12月4日
    
    赞
    
    这是剖析的源码，可以写一个hello world 实际跑一下，打印数据看看吗？
    
    盼盼编程
    
    作者2021年12月4日
    
    赞
    
    直接运行redis就行
    

已无更多数据