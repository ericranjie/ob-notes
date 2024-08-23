# 

点击关注👉 Java面试那些事儿

 _2021年12月04日 11:35_

大家好，我是D哥

点击关注下方公众号，Java面试资料都在这里![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](https://res.wx.qq.com/t/fed_upload/b39ef69e-c4d6-4169-8612-5f00a84860e7/wx-avatar-default.svg)

公众号

### 作者：最怕的其实是孤单  
来源：https://www.jianshu.com/p/f94a0d971b7b  

  

### **# 线程模型1：传统阻塞 I/O 服务模型**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

模型特点：

  

- 采用阻塞IO模式获取输入的数据
    
- 每个链接都需要独立的线程完成数据的输入，业务处理、数据返回。
    

  

问题分析：

  

- 当并发数很大，就会创建大量的线程，占用很大系统资源
    
- 连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read操作，造成线程资源浪费。
    

###   

### **# 线程模型2：Reactor 模式**

  

针对传统阻塞I/O服务模型的2个缺点，解决方案如下：

  

- 基于 I/O 复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理。Reactor对应的叫法: 1. 反应器模式 2. 分发者模式(Dispatcher) 3. 通知者模式(notifier)
    
- 基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。
    

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

#### **# 单 Reactor 单线程**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

模型分析

  

- 优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成
    
- 缺点：性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈
    
- 缺点：可靠性问题，线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障
    
- 使用场景：客户端的数量有限，业务处理非常快速，比如 Redis在业务处理的时间复杂度 O(1) 的情况
    

####   

#### **# 单 Reactor 多线程**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

模型分析

  

- 优点：可以充分的利用多核cpu 的处理能力
    
- 缺点：多线程数据共享和访问比较复杂， reactor 处理所有的事件的监听和响应，在单线程运行， 在高并发场景容易出现性能瓶颈.
    

####   

#### **# 主从 Reactor 多线程**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

模型分析

  

- 优点：父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。
    
- 优点：父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据
    
- 缺点：编程复杂度较高
    
- 结合实例：这种模型在许多项目中广泛使用，包括 Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持
    

##   

## **# 先实现简单的Netty通信**

  

### 服务端示例

```
public static void main(String[] args) {
```

  

### 客户端示例

```
public static void main(String[] args) {
```

  

快启动试试看把，不过需要注意的是，得先启动服务端哦~

  

## **# SpringBoot + Netty4实现rpc框架**

> 好了，接下来就让我们进入正题，让我们利用我们所学的知识去实现自己一个简单的rpc框架吧

简单说下RPC（Remote Procedure Call）远程过程调用，简单的理解是一个节点请求另一个节点提供的服务。让两个服务之间调用就像调用本地方法一样。

  

RPC时序图：

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

RPC流程：

> 1. 【客户端】发起调用
>     
> 2. 【客户端】数据编码
>     
> 3. 【客户端】发送编码后的数据到服务端
>     
> 4. 【服务端】接收客户端发送的数据
>     
> 5. 【服务端】对数据进行解码
>     
> 6. 【服务端】处理消息业务并返回结果值
>     
> 7. 【服务端】对结果值编码
>     
> 8. 【服务端】将编码后的结果值回传给客户端
>     
> 9. 【客户端】接收结果值
>     
> 10. 【客户端】解码结果值
>     
> 11. 【客户端】处理返回数据业务
>     

### **# 引入依赖**

```
<dependencies>
```

### **# 编写服务端**

  

自定义消息协议：

```
/**
```

  

自定义Rpc注解：

```
/**
```

  

定义ServerHandle业务处理器：

```
/**
```

  

定义NettyServer端：

```
/**
```

  

自定义rpc配置属性类：

```
/**
```

  

创建Server端启动配置类：

```
/**
```

  

**# 注入Spring容器**

  

此时有两种方式让该配置自动注入Spring容器生效：

  

1. 自动注入
    
    > 在resource目录下创建META-INF目录，创建spring.factories文件
    > 
    > 在该文件里写上
    > 
    > org.springframework.boot.autoconfigure.EnableAutoConfiguration=${包路径:xxx.xxx.xxx}.${配置类：ServerBeanConfig}
    > 
    > 配置好之后，在SpringBoot启动时会自动加载该配置类。
    
2. 通过注解注入
    
    > /**  
    >  * 自定义SpringBoot启动注解  
    >  * 注入ServerBeanConfig配置类  
    >  *  
    >  * @author ZC  
    >  * @date 2021/3/1 23:48  
    >  */  
    > @Target({ElementType.TYPE})  
    > @Retention(RetentionPolicy.RUNTIME)  
    > @Documented  
    > @Inherited  
    > @ImportAutoConfiguration({ServerBeanConfig.class})  
    > public @interface EnableNettyServer {  
    > }
    

### **# 编写客户端**  

  

创建客户端处理器`ClientHandle

```
/**
```

  

创建客户端启动类NettyClient

```
/**
```

  

定义Netty客户端Bean后置处理器

```
/**
```

  

定义客户端配置类

```
/**
```

  

最后和服务端一样，注入Spring容器

```
/**
```

  

至此我们的SpringBoot + Netty4的就已经实现了最最简单的rpc框架模式了；然后我们就可以引用我们自己的rpc依赖了。

  

最后再执行一下maven命令

```
mvn install
```

  

## **# netty-rpc-examples例子**

  

### 接口服务

  

pom里啥也没有。。。

  

定义一个接口

```
/**
```

  

### **# rpc-server服务端**

> 正常的SpringBoot工程

引入pom

```
<!-- 自定义rpc依赖 -->
```

  

配置属性

```
# 应用名称
```

  

创建一个实体类

```
/**
```

  

创建Server实现Test1Api接口

```
/**
```

  

最后在SpringBoot启动类上加上@EnableNettyServer

```
/**
```

  

### **# rpc-server客户端**

  

引入pom依赖

```
<dependency>
```

  

创建Controller

```
/**
```

  

最后在启动类上加上注解@EnableNettyClient

```
@EnableNettyClient
```

> 先运行服务端，在运行客户端，然后在调用客户端接口就可以看到服务端能够接收到客户端发来的消息，然后服务端处理并返回，客户端接收并返回。。。
> 
> 至此，一个小demo就完成了。
> 
> 当然啦，后续还有很多需求需要处理的，比方说当前demo中客户端每次通信都需要创建一个实例去连接、服务的注册、客户端和服务端是同一个应用等等，这个后面再慢慢完善吧  

  

  

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)技术交流群![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)**

  

D哥也了一个技术群，主要针对一些新的技术和开源项目值不值得去研究和IDEA使用的“骚操作”，有兴趣入群的同学，可以长扫描区域二维码，一定要注意事项：**城市+昵称+技术方向**，根据格式备注，可快速通过。

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

▲长按扫描

  

**![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)热门推荐：**

- [知乎高赞：王小波的计算机水平有多好？](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247562789&idx=1&sn=cffd4e450c75ac2a3c0a2a064e62eaa2&chksm=e8fc6a2cdf8be33ac700cc2a0020941cd47f86a76e68904d3744d3ec2a1f6c76b38517bc49a3&scene=21#wechat_redirect)
    
- [如何防止你的 jar 被反编译？](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247562789&idx=2&sn=29b74e756805008f1d45c50d024c1b30&chksm=e8fc6a2cdf8be33af1e4d21d130449bef9d389ed5005c8e3d06214d3659604b20a261d4b8fd0&scene=21#wechat_redirect)
    
- [推荐一款开源java版的视频管理系统！(附源码)](http://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247562789&idx=3&sn=69d24b278968fb720c414bbc6b3a3475&chksm=e8fc6a2cdf8be33a408f4f1504ce37c5bc6ffda588d502c1332bb9c0374e0c1ef0681c3ed045&scene=21#wechat_redirect)  
    
      
    

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Java899

Java · 目录

上一篇现代API渗透技术，我服了...下一篇美团一面：说说前、后端分离权限控制设计和实现思路？

阅读 3279

​