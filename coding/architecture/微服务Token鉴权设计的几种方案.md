捡田螺的小男孩

_2024年04月02日 08:35_ _广东_

来源：juejin.cn/post/7329352197837029385

- Token透传（不推荐）

- Fegin内部调用方式

- Dubbo内部调用方式

- Spring Boot Web + Dubbo内部调用方式

- 常规模式

- 与K8S集成

______________________________________________________________________

## **Token透传（不推荐）**

刚开始接触微服务时网上给的方案大都数是通过透传Token做鉴权，但我认为这种方式不是很妥当。接着往下看：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/knmrNHnmCLFtibwEnldja6mKu0rcicibhu3yMTw0OjMibaG6iaY754No3ibsRkdq16mEcJs65LRZYpWj5Bh5ucDRSwUg/640?wx_fmt=jpeg&from=appmsg&wxfrom=13)

图片

这种方式通过透传Token使得各微服务都能获取到当前登录人信息，在代码编写上确实可能会方便，但我认为这不是一种很好的设计方式。

原因一：内部API与外部API混合在一起不太好区分。

原因二：内部调用的微服务API因该具备无状态性质，这样才能保证方法的原子性以提高代码复用率。

换句话说：B服务提供API时不因该关心当前是否为登录状态，登录状态应该由路由中的第一个服务校验维护，在调用后续服务时应该显示的传入相关参数。比如以下场景：

场景一：用户签到添加积分

场景二：后台管理员给用户手动添加积分

场景三：分布式调度批量增加用户积分

根据需求积分服务提供了一个给用户添加积分的API，如果你的API是通过获取的当前登录用户ID增加的积分，那么面对场景二时你需要重新编写一个给用户添加积分的API，因为当前登录的是后台管理员而不是用户（代码复用率较低）

**不透传数据，显示的提供入参**

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

路由到达的第一个服务已经对Token进行了解析认证并将userId显示的传递给了后续服务，后续服务不需要再对token进行解析认证。根据1.1的三个场景只需要提供一个入参包含userId的API，保证了函数的原子性提供代码复用率。

**注意: 提供的API不能暴露给外网，我们需要在路径上做区分，避免外网非法访问内部API。我们可以订好内部调用API路径规则，如：** **`/api/inside/\*\*` 。在网关层拒绝内部调用API请求的访问。**

#### **统一授权**

统一授权是指：将API鉴权集中在应用网关上

## **Fegin内部调用方式**

Spring Cloud Gateway + Fegin内部调用，集中在Gateway上做统一认证鉴权，鉴权后在请求头中添加鉴权后的信息转发给后续服务，如：userId等。。。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

缺点：A服务调用B服务时，B服务需要写一个内部调用的Controller接口A服务才能通过Fegin调用到B服务，增加了代码量（这里的设计方案是内部调用与外部调用Controller是分开的）

## **Dubbo内部调用方式**

Spring Cloud Gateway + Dubbo内部调用，集中在Gateway上做统一认证鉴权，鉴权后在请求头中添加鉴权后的信息转发给后续服务，如：userId等。。。

优点：与第一种相比不需要额外编写一个Controller接口，只有本地service与远程DubboService的区别，代码更简洁。

缺点：项目技术栈略微增加了复杂度。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

## **Spring Boot Web + Dubbo内部调用方式**

这里的设计方案直接去掉了Gateway，直接使用了一个Spring Boot Web项目来代替Gateway。但需要注意的是应该将Web项目的容器换成Undertow，因为Tomcat是阻塞式的容器，不换也不是不行，但吞吐量可能会少一下，Undertow是非阻塞式的容器，可以与Gateway到达相同的效果。（非阻塞式：当请求为线程进入阻塞状态时，当前线程会被挂起，当前的计算资源会去做别的事情，当被挂起的线程收到响应时才会被继续执行，压榨CPU用更少的资源做更多的事情，但并不会提升性能）

因为去掉了Gateway我们需要将所有服务的Controller集成到Web应用，然后在这个Web应用上做统一认证授权。如果将所有代码写到Web应用中，这样可能不合适，我们可以选择每个服务创建一个Controller模块，Web网关服务只有一个启动类，通过依赖的方式集成所有服务的Controller。

优点：简化了项目结构，所有服务只有service代码。性能压测时不用考虑Gateway的线程池使用情况，业务服务只需要考虑Dubbo线程池的使用情况。

缺点：没办法通过配置中心动态调整路由。比如说增加了一个服务Gateway可以不重启通过配置中心增加路由配置即可。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

#### **非统一授权**

非统一授权：不在应用网关上集成鉴权，网关只有单一的路由转发业务。各位服务都有自己的鉴权方式，当然也可以通过jar包的方式统一各服务的鉴权方式。

## **常规模式**

通过编写通用的鉴权模块，各服务集成该模块。该模块具备以下功能：

1. JWT Token解析

1. 权限校验拦截

1. 缓存（本地缓存\\Redis缓存）

这种模式更适合大型项目团队，可能各微服务都由一个项目组负责。各服务维护自己的权限规则（这里指的是权限规则数据，规则是统一的）

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

该模式下由于应用网关比较轻量级，不再涉及复杂的鉴权流程，使得项目部署可以更灵活，当我们使用K8S部署项目时，我们可以将应用网关替换成K8S中的Ingress网关。

我们先看常规模式部署在K8S中完整的链路:

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

当用户访问时会先到达K8S Ingress网关通过应用网关Service的负载均衡调用应用网关，应用网关需要通过注册中心获取服务注册列表，通过服务注册列表负载均衡到后续服务。

## **与K8S集成**

我们再来看看将应用网关替换成K8S中的Ingress网关的完整链路：

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

图片

这里不仅只是去掉了应用网关，同时我们通过K8S Service 负载均衡的能力去掉了注册中心。减少了我们部署微服务时还要额外搭建一套注册中心。同时减少了一层没必要的转发。至于K8S中的Service，大家可以理解成一个本地的host假域名，比如我们在K8S给商品创建一个Service，名称为：goods-svc。那么我们可以通过goods-svc直连。如：

1. `http://goods-svc:8080/api/goods/info/10001`

1. `dubbo://goods-svc:20880`

**方案没有对错，选择适合自己的就是最好的。**

End

![](http://mmbiz.qpic.cn/mmbiz_png/m2jCBpUlqWSUp1N5WBmiaHA6yGicBmUTfv255ZW1ZnJTIRTcuPlbXhqS5MkrlgGicESS3VfZicCBRibxXIrbVAAk52g/300?wx_fmt=png&wxfrom=19)

**田螺程序人生**

主要聊程序员职业生涯~~

4篇原创内容

公众号

![](http://mmbiz.qpic.cn/mmbiz_png/iaPU220ia3N7QfHsbKk3mGa1lsrNh9kID5jJsopIGBnric9v4xKcFOv50y6N3A3CVRteuJ9tQI0IAIh37R3dpvGog/300?wx_fmt=png&wxfrom=19)

**程序员田螺**

工作多年，呆过几个公司，给大家分享职场的经验和教训，帮新人避坑~~

64篇原创内容

公众号

阅读 2999

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/cEDP0gXG22n22jFlEqLy1BBlJ0yJXicTGDibTmjqib1j2yZKibo0BicvWYKToCChBqIEJFSdyF2DR5Wya71XCrHTPnQ/300?wx_fmt=png&wxfrom=18)

捡田螺的小男孩

92383

写留言

写留言

**留言**

暂无留言
