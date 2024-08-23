

原创 悟空聊架构 悟空聊架构

 _2022年01月14日 11:24_

![图片](https://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ1y2VMsAyjEFy5I9m8UZbxPicB337bjKdsziavtqHUVuXkAzxJSZsCOHiaFaEUAbGZpDhGAwxiaAdFJtg/640?wx_fmt=png&wxfrom=13&tp=wxpic)

这是悟空的第 85 篇原创文章  

官网：www.passjava.cn

  

大家好，我是悟空呀。

上次我们深入讲解了 [Ribbon 的架构原理](https://mp.weixin.qq.com/s?__biz=MzAwMjI0ODk0NA==&mid=2451961128&idx=1&sn=3e2c399aa8ac70d9bdd2df04ccca61db&scene=21#wechat_redirect)，这次我们再来看下 Feign 远程调用的架构原理。

![图片](https://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ1y2VMsAyjEFy5I9m8UZbxPvFXrdZGVM4x3IwkRSFZvX9oYSBSPZAlPGVC0xD5arXTaNDtZvxPWcw/640?wx_fmt=png&wxfrom=13&tp=wxpic)  

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=19)

**悟空聊架构**

用故事讲解分布式、架构。 《 JVM 性能调优实战》专栏作者， 《Spring Cloud 实战 PassJava》开源作者， 自主开发了 PMP 刷题小程序。

205篇原创内容

公众号

## 一、理解远程调用

远程调用怎么理解呢？

`远程调用`和`本地调用`是相对的，那我们先说本地调用更好理解些，本地调用就是同一个 Service 里面的方法 A 调用方法 B。

那远程调用就是不同 Service 之间的方法调用。Service 级的方法调用，就是我们自己构造请求 URL和请求参数，就可以发起远程调用了。

在服务之间调用的话，我们都是基于 HTTP 协议，一般用到的远程服务框架有 OKHttp3，Netty, HttpURLConnection 等。其调用流程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ1y2VMsAyjEFy5I9m8UZbxPpGwzw74ic32u5sxAAmmlxCeOpiccpclcrxgJx00vb3YyhJNicgAOicfuUg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

但是这种虚线方框中的构造请求的过程是很繁琐的，有没有更简便的方式呢？

**Feign** 就是来简化我们发起远程调用的代码的，那简化到什么程度呢？**简化成就像调用本地方法那样简单。**

比如我的开源项目 PassJava 中的使用 Feign 执行远程调用的代码：

`//远程调用拿到该用户的学习时长   R memberStudyTimeList = studyTimeFeignService.getMemberStudyTimeListTest(id);   `

而 Feign 又是 Spring Cloud 微服务技术栈中非常重要的一个组件，如果让你来设计这个微服务组件，你会怎么来设计呢？

我们需要考虑这几个因素：

- 如何使远程调用像本地方法调用简单？
    
- Feign 如何找到远程服务的地址的？
    
- Feign 是如何进行负载均衡的？
    

接下来我们围绕这些核心问题来一起看下 Feign 的设计原理。

## 二、Feign 和 OpenFeign

OpenFeign 组件的前身是 Netflix Feign 项目，它最早是作为 Netflix OSS 项目的一部分，由 Netflix 公司开发。后来 Feign 项目被贡献给了开源组织，于是才有了我们今天使用的 Spring Cloud OpenFeign 组件。

Feign 和 OpenFeign 有很多大同小异之处，不同的是 OpenFeign 支持 MVC 注解。

可以认为 OpenFeign 为 Feign 的增强版。

简单总结下 OpenFeign 能用来做什么：

- OpenFeign 是声明式的 HTTP 客户端，让远程调用更简单。
    
- 提供了HTTP请求的模板，编写简单的接口和插入注解，就可以定义好HTTP请求的参数、格式、地址等信息
    
- 整合了Ribbon（负载均衡组件）和 Hystix（服务熔断组件），不需要显示使用这两个组件
    
- Spring Cloud Feign 在 Netflix Feign的基础上扩展了对SpringMVC注解的支持
    

## 三、OpenFeign 如何用？

OpenFeign 的使用也很简单，这里还是用我的开源 SpringCloud 项目 PassJava 作为示例。

> 开源地址: https://github.com/Jackson0714/PassJava-Platform
> 
> 喜欢的小伙伴来点个 Star 吧，冲 2K Star。

Member 服务远程调用 Study 服务的方法 memberStudyTime()，如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ1y2VMsAyjEFy5I9m8UZbxPSq65JIUUxO3ibzAyxBOg2xxfFKqjxxg7GvRCgECOiaFcLYEIHKBB94tA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

第一步：Member 服务需要定义一个 OpenFeign 接口：

`@FeignClient("passjava-study")   public interface StudyTimeFeignService {       @RequestMapping("study/studytime/member/list/test/{id}")       public R getMemberStudyTimeListTest(@PathVariable("id") Long id);   }   `

我们可以看到这个 interface 上添加了注解`@FeignClient`，而且括号里面指定了服务名：passjava-study。显示声明这个接口用来远程调用 `passjava-study`服务。

第二步：Member 启动类上添加 `@EnableFeignClients`注解开启远程调用服务，且需要开启服务发现。如下所示：

`@EnableFeignClients(basePackages = "com.jackson0714.passjava.member.feign")   @EnableDiscoveryClient   `

第三步：Study 服务定义一个方法，其方法路径和 Member 服务中的接口 URL 地址一致即可。

URL 地址："study/studytime/member/list/test/{id}"

`@RestController   @RequestMapping("study/studytime")   public class StudyTimeController {       @RequestMapping("/member/list/test/{id}")       public R memberStudyTimeTest(@PathVariable("id") Long id) {          ...        }   }   `

第四步：Member 服务的 POM 文件中引入 OpenFeign 组件。

`<dependency>       <groupId>org.springframework.cloud</groupId>       <artifactId>spring-cloud-starter-openfeign</artifactId>   </dependency>   `

第五步：引入 studyTimeFeignService，Member 服务远程调用 Study 服务即可。

`Autowired   private StudyTimeFeignService studyTimeFeignService;      studyTimeFeignService.getMemberStudyTimeListTest(id);   `

通过上面的示例，我们知道，加了 @FeignClient 注解的接口后，我们就可以调用它定义的接口，然后就可以调用到远程服务了。

> 这里你是否有疑问：为什么接口都没有实现，就可以调用了？

OpenFeign 使用起来倒是简单，但是里面的原理可没有那么简单，OpenFeign 帮我们做了很多事情，接下来我们来看下 OpenFeign 的架构原理。

## 四、梳理 OpenFeign 的核心流程

先看下 OpenFeign 的核心流程图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 1、在 Spring 项目启动阶段，服务 A 的OpenFeign 框架会发起一个主动的扫包流程。
    
- 2、从指定的目录下扫描并加载所有被 @FeignClient 注解修饰的接口，然后将这些接口转换成 Bean，统一交给 Spring 来管理。
    
- 3、根据这些接口会经过 MVC Contract 协议解析，将方法上的注解都解析出来，放到 MethodMetadata 元数据中。
    
- 4、基于上面加载的每一个 FeignClient 接口，会生成一个动态代理对象，指向了一个包含对应方法的 MethodHandler 的 HashMap。MethodHandler 对元数据有引用关系。生成的动态代理对象会被添加到 Spring 容器中，并注入到对应的服务里。
    
- 5、服务 A 调用接口，准备发起远程调用。
    
- 6、从动态代理对象 Proxy 中找到一个 MethodHandler 实例，生成 Request，包含有服务的请求 URL（不包含服务的 IP）。
    
- 7、经过负载均衡算法找到一个服务的 IP 地址，拼接出请求的 URL
    
- 8、服务 B 处理服务 A 发起的远程调用请求，执行业务逻辑后，返回响应给服务 A。
    

针对上面的流程，我们再来看下每一步的设计原理。首先主动扫包是如何扫的呢？

## 五、OpeFeign 包扫描原理

上面的 PassJava 示例代码中，涉及到了一个 OpenFeign 的注解：`@EnableFeignClients`。根据字面意思可以知道，可以注解是开启 OpenFeign 功能的。

包扫描的基本流程如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

（1）@EnableFeignClients 这个注解使用 Spring 框架的 `Import` 注解导入了 FeignClientsRegistrar 类，开始了 OpenFeign 组件的加载。PassJava 示例代码如下所示。

`// 启动类加上这个注解   @EnableFeignClients(basePackages = "com.jackson0714.passjava.member.feign")      // EnableFeignClients 类还引入了 FeignClientsRegistrar 类   @Import(FeignClientsRegistrar.class)   public @interface EnableFeignClients {     ...   }   `

（2）FeignClientsRegistrar 负责 Feign 接口的加载。

源码如下所示：

`@Override   public void registerBeanDefinitions(AnnotationMetadata metadata,         BeanDefinitionRegistry registry) {      // 注册配置      registerDefaultConfiguration(metadata, registry);      // 注册 FeignClient      registerFeignClients(metadata, registry);   }   `

（3）registerFeignClients 会扫描指定包。

核心源码如下，调用 find 方法来查找指定路径 basePackage 的所有带有 @FeignClients 注解的带有 @FeignClient 注解的类、接口。

`Set<BeanDefinition> candidateComponents = scanner         .findCandidateComponents(basePackage);   `

（4）只保留带有 @FeignClient 的接口。

`// 判断是否是带有注解的 Bean。   if (candidateComponent instanceof AnnotatedBeanDefinition) {     // 判断是否是接口      AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;      AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();     // @FeignClient 只能指定在接口上。      Assert.isTrue(annotationMetadata.isInterface(),            "@FeignClient can only be specified on an interface");   `

接下来我们再来看这些扫描到的接口是如何注册到 Spring 中。

## 六、注册 FeignClient 到 Spring 的原理

还是在 registerFeignClients 方法中，当 FeignClient 扫描完后，就要为这些 FeignClient 接口生成一个动态代理对象。

顺藤摸瓜，进到这个方法里面，可以看到这一段代码：

`BeanDefinitionBuilder definition = BeanDefinitionBuilder         .genericBeanDefinition(FeignClientFactoryBean.class);   `

核心就是 FeignClientFactoryBean 类，根据类的名字我们可以知道这是一个工厂类，用来创建 FeignClient Bean 的。

我们最开始用的 @FeignClient，里面有个参数 "passjava-study"，这个是注解的属性，当 OpenFeign 框架去创建 FeignClient Bean 的时候，就会使用这些参数去生成 Bean。流程图如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 解析 `@FeignClient` 定义的属性。
    
- 将注解`@FeignClient` 的属性 +  接口 `StudyTimeFeignService`的信息构造成一个 StudyTimeFeignService 的 beanDefinition。
    
- 然后将 beanDefinition 转换成一个 holder，这个 holder 就是包含了 beanDefinition, alias, beanName 信息。
    
- 最后将这个 holder 注册到 Spring 容器中。
    

源码如下：

`// 生成 beanDefinition   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();   // 转换成 holder，包含了 beanDefinition, alias, beanName 信息   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,       new String[] { alias });   // 注册到 Spring 上下文中。   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);   `

上面我们已经知道 FeignClient  的接口是如何注册到 Spring 容器中了。后面服务要调用接口的时候，就可以直接用 FeignClient 的接口方法了，如下所示：

`@Autowired   private StudyTimeFeignService studyTimeFeignService;      // 省略部分代码   // 直接调用    studyTimeFeignService.getMemberStudyTimeListTest(id);   `

但是我们并没有细讲这个 FeignClient 的创建细节，下面我们看下 FeignClient 的创建细节，这个也是 OpenFeign 核心原理。

## 七、OpenFeign 动态代理原理

上面的源码解析中我们也提到了是由这个工厂类  FeignClientFactoryBean 来创建 FeignCient Bean，所以我们有必要对这个类进行剖析。

在创建 FeignClient Bean 的过程中就会去生成动态代理对象。调用接口时，其实就是调用动态代理对象的方法来发起请求的。

分析动态代理的入口方法为 getObject()。源码如下所示：

`Targeter targeter = get(context, Targeter.class);   return (T) targeter.target(this, builder, context,         new HardCodedTarget<>(this.type, this.name, url));   `

接着调用 target 方法这一块，里面的代码真的很多很细，我把核心的代码拿出来给大家讲下，这个 target 会有两种实现类：

DefaultTargeter 和 HystrixTargeter。而不论是哪种 target，都需要去调用 Feign.java 的 builder 方法去构造一个 feign client。

在构造的过程中，依赖 ReflectiveFeign 去构造。源码如下：

`// 省略部分代码   public class ReflectiveFeign extends Feign {     // 为 feign client 接口中的每个接口方法创建一个 methodHandler    public <T> T newInstance(Target<T> target) {       for(...) {         methodToHandler.put(method, handler);       }       // 基于 JDK 动态代理的机制，创建了一个 passjava-study 接口的动态代理，所有对接口的调用都会被拦截，然后转交给 handler 的方法。       InvocationHandler handler = factory.create(target, methodToHandler);       T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),             new Class<?>[] {target.type()}, handler);   }   `

> ReflectiveFeign 做的工作就是为带有 @FeignClient 注解的接口，创建出接口方法的动态代理对象。

比如示例代码中的接口 StudyTimeFeignService，会给这个接口中的方法 getMemberStudyTimeList 创建一个动态代理对象。

`@FeignClient("passjava-study")   public interface StudyTimeFeignService {       @RequestMapping("study/studytime/member/list/test/{id}")       public R getMemberStudyTimeList(@PathVariable("id") Long id);   }   `

创建动态代理的原理图如下所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 解析 FeignClient 接口上各个方法级别的注解，比如远程接口的 URL、接口类型（Get、Post 等）、各个请求参数等。这里用到了 MVC Contract 协议解析，后面会讲到。
    
- 然后将解析到的数据封装成元数据，并为每一个方法生成一个对应的 MethodHandler 类作为方法级别的代理。相当于把服务的请求地址、接口类型等都帮我们封装好了。这些 MethodHandler 方法会放到一个 HashMap 中。
    
- 然后会生成一个 InvocationHandler 用来管理这个 hashMap，其中 Dispatch 指向这个 HashMap。
    
- 然后使用 Java 的 JDK 原生的动态代理，实现了 FeignClient 接口的动态代理 Proxy 对象。这个 Proxy 会添加到 Spring 容器中。
    
- 当要调用接口方法时，其实会调用动态代理 Proxy 对象的 methodHandler 来发送请求。
    

这个动态代理对象的结构如下所示，它包含了所有接口方法的 MethodHandler。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## 八、解析 MVC 注解的原理

上面我们讲到了接口上是有一些注解的，比如 @RequestMapping，@PathVariable，这些注解统称为 Spring MVC 注解。但是由于 OpenFeign 是不理解这些注解的，所以需要进行一次解析。

解析的流程图如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

而解析的类就是 `SpringMvcContract` 类，调用 `parseAndValidateMetadata`  进行解析。解析完之后，就会生成元数据列表。源码如下所示：

`List<MethodMetadata> metadata = contract.parseAndValidateMetadata(target.type());   `

这个类在这个路径下，大家可以自行翻阅下如何解析的，不在本篇的讨论范围内。

`https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/support/SpringMvcContract.java   `

这个元数据 MethodMetadata 里面有什么东西呢？

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 方法的定义，如 StudyTimeFeignService 的 getMemberStudyTimeList 方法。
    
- 方法的参数类型，如 Long。
    
- 发送 HTTP 请求的地址，如 /study/studytime/member/list/test/{id}。
    

然后每个接口方法就会有对应的一个 MethodHandler，它里面就包含了元数据，当我们调用接口方法时，其实是调用动态代理对象的 MethodHandler 来发送远程调用请求的。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面我们针对 OpenFeign 框架如何为 FeignClient 接口生成动态代理已经讲完了，下面我们再来看下当我们调用接口方法时，动态代理对象是如何发送远程调用请求的。

## 九、OpenFeign 发送请求的原理

先上流程图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

还是在 ReflectiveFeign 类中，有一个 invoke 方法，会执行以下代码：

`dispatch.get(method).invoke(args);   `

这个 dispatch 我们之前已经讲解过了，它指向了一个 HashMap，里面包含了 FeignClient 每个接口的 MethodHandler 类。

这行代码的意思就是根据 method 找到 MethodHandler，调用它的 invoke 方法，并且传的参数就是我们接口中的定义的参数。

那我们再跟进去看下这个 MethodHandler invoke 方法里面做了什么事情。源码如下所示：

`public Object invoke(Object[] argv) throws Throwable {     RequestTemplate template = buildTemplateFromArgs.create(argv);     ...   }   `

我们可以看到这个方法里面生成了 RequestTemplate，它的值类似如下：

`GET /study/list/test/1 HTTP/1.1   `

RequestTemplate 转换成 Request，它的值类似如下：

`GET http://passjava-study/study/list/test/1 HTTP/1.1   `

这路径不就是我们要 study 服务的方法，这样就可以直接调用到 study 服了呀！

OpenFeign 帮我们组装好了发起远程调用的一切，我们只管调用就好了。

接着 MethodHandler 会执行以下方法，发起 HTTP 请求。

`client.execute(request, options);   `

从上面的我们要调用的服务就是 passjava-study，但是这个服务的具体 IP 地址我们是不知道的，那 OpenFeign 是如何获取到 passjava-study 服务的 IP 地址的呢？

回想下最开始我们提出的核心问题：OpenFeign 是如何进行负载均衡的？

我们是否可以联想到上一讲的 Ribbon 负载均衡，它不就是用来做 IP 地址选择的么？

那我们就来看下 OpenFeign 又是如何和 Ribbon 进行整合的。

## 十、OpenFeign 如何与 Ribbon 整合的原理

为了验证 Ribbon 的负载均衡，我们需要启动两个 passjava-study 服务，这里我启动了两个服务，端口号分别为 12100 和 12200，IP 地址都是本机 IP：192.168.10.197。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接着上面的源码继续看，client.execute() 方法其实会调用 LoadBalancerFeignClient 的 exceute 方法。

这个方法里面的执行流程如下图所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 将服务名称 `passjava-study` 从 Request 的 URL 中删掉，剩下的如下所示：
    

`GET http:///study/list/test/1 HTTP/1.1   `

- 根据服务名从缓存中找 FeignLoadBalancer，如果缓存中没有，则创建一个 FeignLoadBalancer。
    
- FeignLoadBalancer 会创建出一个 command，这个 command 会执行一个 sumbit 方法。
    
- submit 方法里面就会用 Ribbon 的负载均衡算法选择一个 server。源码如下：
    

`Server svc = lb.chooseServer(loadBalancerKey);   `

通过 debug 调试，我们可以看到两次请求的端口号不一样，一个是 12200，一个是 12100，说明确实进行了负载均衡。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

- 然后将 IP 地址和之前剔除掉服务名称的 URL 进行拼接，生成最后的服务地址。
    
- 最后 FeignLoadBalancer 执行 execute 方法发送请求。
    

**那大家有没有疑问，Ribbon 是如何拿到服务地址列表的**？这个就是上一讲 Ribbon 架构里面的内容。

Ribbon 的核心组件 ServerListUpdater，用来同步注册表的，它有一个实现类 PollingServerListUpdater ，专门用来做定时同步的。默认1s 后执行一个 Runnable 线程，后面就是每隔 30s 执行 Runnable 线程。这个 Runnable 线程就是去获取注册中心的注册表的。

## 十一、OpenFeign 处理响应的原理

当远程服务 passjava-study 处理完业务逻辑后，就会返回 reponse 给 passjava-member 服务了，这里还会对 reponse 进行一次解码操作。

`Object result = decode(response);   `

这个里面做的事情就是调用 ResponseEntityDecoder 的 decode 方法，将 Json 字符串转化为 Bean 对象。

## 十二、总结

本文通过我的开源项目 PassJava 中用到的 OpenFeign 作为示例代码作为入口进行讲解。然后以图解+解读源码的方式深入剖析了 OpenFeign 的运行机制和架构设计。

**核心思想：**

- OpenFeign 会扫描带有 @FeignClient 注解的接口，然后为其生成一个动态代理。
    
- 动态代理里面包含有接口方法的 MethodHandler，MethodHandler 里面又包含经过 MVC Contract 解析注解后的元数据。
    
- 发起请求时，MethodHandler 会生成一个 Request。
    
- 负载均衡器 Ribbon 会从服务列表中选取一个 Server，拿到对应的 IP 地址后，拼接成最后的 URL，就可以发起远程服务调用了。
    

OpenFeign 的核心流程图：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=19)

**悟空聊架构**

用故事讲解分布式、架构。 《 JVM 性能调优实战》专栏作者， 《Spring Cloud 实战 PassJava》开源作者， 自主开发了 PMP 刷题小程序。

205篇原创内容

公众号

- END -

写了两本 PDF，回复 分布式 或 PDF 下载。

我的 JVM 专栏已上架，回复 JVM 领取

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

_**我是悟空，努力变强，变身超级赛亚人！**_

![](https://mmbiz.qlogo.cn/mmbiz_jpg/VIuZAhPBd2Bx2XnAXUXmSmITibF71jIEDWyhVHJGUwFGDTVbAaP13vnJWcC9cYspRJCezA0phkcB3WnDdfsSqGw/0?wx_fmt=jpeg)

悟空聊架构

 加鸡腿🍗 

![赞赏二维码](https://mp.weixin.qq.com/s?__biz=MzAwMjI0ODk0NA==&mid=2451961986&idx=1&sn=f805fc10fd08353f2b1916ae0beec5fd&chksm=8d1c011dba6b880b1bf946f73d85f180011aea6f2f732167789afafcc249aa5a8d9f520ef91c&mpshare=1&scene=24&srcid=0114zcAn7VLiGdoU3JeV33sz&sharer_sharetime=1642133356118&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d08ba961b0cbc03ad8fbc2c05e11a3805c62eeddf4ef9941580c3db551f5c2b5544b5edb8ac6da8019dcbe424a3a63419ab247d66b086aec23963a187590ea70857119b5c4e5e672ab748b8a7388e9780dfb797a60a9c4664e831a4c04dd830b9ecad672de0ce384029f5648b54b482e8ddd5185e2280b11b1&ascene=0&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=Windows+11+x64&version=63090b19&lang=zh_CN&countrycode=CN&exportkey=n_ChQIAhIQQA1PDY12BKxKJeEul3sW4hLZAQIE97dBBAEAAAAAAATzF6YSYIwAAAAOpnltbLcz9gKNyK89dVj0nPdz86qS4qtuLMRC%2FCLkBYHntVLjyK%2FtmmEBHGGnNkj%2F%2FruD955Ik1AMsjVguuiR5s2k9AZ3peT%2FAJfrszgfiOjVX7C3xnB%2BU9%2BvB%2BQfnr5HKgIH8GyZamXpNlOJFwfXqm%2BZLYO4w6jufmlbnbrgAH47PoQ7Xg3oQBmBzA%2FRRMGiHjnqdG1ZOjx8Imfp9geOs6aYRd9H1T%2F2Y9A7%2Fg3Sa%2FtNnCzeBx54ZkFNmIqpaiFgHuw%3D&acctmode=0&pass_ticket=ynT7PFwikFrURtKafQO8pZ8%2FjTUmrE3YEloJF%2FY2%2BoeT5D%2FwQD194TRq2QcEAP09&wx_header=1&fasttmpl_type=0&fasttmpl_fullversion=7350504-zh_CN-zip&fasttmpl_flag=1)喜欢作者

5人喜欢

![](http://wx.qlogo.cn/mmopen/Q3auHgzwzM6uj3Iy0G3tvlF2hqicMicOXQxFggbQv5aW0fia5LrwSMXde4j1nkGBr656pLeS5Kw0l3ngXRiclP0cMBgaEzNu3WJOaYjLD2lUcjs/64)![](http://wx.qlogo.cn/mmopen/qibXTFhPic1vOcPOx4lJnhZ94pPa3YJfZoibAyd4ABtlJwe8yQcbbEUpbtaAWCJqCicRCJzUzaC3xicOmgbehnOsdIY2uolkDegsHia0bb8lfjlLPt5Cll4tLvXrxboply6NLia/64)![](http://wx.qlogo.cn/mmopen/ajNVdqHZLLCU6q3EJgkGdSc3CyWkSNMcSwTvFponibVsiczIyW0ibzPsTIWSicMQ0GaicWKKk0Y7jrXJrAE09S9w9bQ/64)![](http://wx.qlogo.cn/mmopen/PiajxSqBRaELgNr5DLl70MkFHBEsGHx6t39bc7cLzbHDTomh8za1uZ9UOEJY8aMN1Sh1wTu870kg1ov1dKibKBF2snTYNKsZSYDMwNr6D8QD06PWicVB1KfXxXibOs547p3J/64)![](http://wx.qlogo.cn/mmopen/qibXTFhPic1vPwA2pjODDAe2pQ6oQehM34ePbrPtuHHYfXlCkGMmib1BTYr5Sj0icudibYEU46cT4iaaRpLbEcBJyotWzEzjAR9ib0X/64)

SpringCloud 源码+架构剖析19

SpringCloud 源码+架构剖析 · 目录

上一篇16 图 | 实战 Eureka 集群搭建+服务注册+调用下一篇6000 字｜20 图｜Nacos 手摸手教程

阅读原文

阅读 3741

​

写留言

**留言 46**

- 悟空呀
    
    2022年1月14日
    
    赞6
    
    这篇理论性很强，图画了很久，求个在看，老铁们![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- 悟空呀
    
    2022年1月15日
    
    赞1
    
    补充：JDK 动态代理可以理解为动态生成了一个匿名类，然后实现了 FeignClient 接口，基于这个匿名类，创建了一个对象，也就是 proxy。对接口的调用都转交给动态代理对象处理了。
    
    置顶
    
- 一枝花算不算浪漫
    
    2022年1月14日
    
    赞2
    
    卷王之王![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞3
    
    终于卷出来了，全网最细应该不过分吧![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    一枝花算不算浪漫
    
    2022年1月14日
    
    赞5
    
    不过分 这次全网都知道你最细了![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 薛定谔的萌虎
    
    2022年1月14日
    
    赞2
    
    硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞3
    
    那必须的，毕竟写了这么久
    
- 麦田的守望者
    
    2022年1月14日
    
    赞2
    
    先收藏，慢慢研究
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    Ribbon 和 Eurkea 的也一起收藏下吧![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 田先森😎
    
    2022年1月15日
    
    赞1
    
    坐等大佬的Hystrix![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月15日
    
    赞1
    
    催更了![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Cloud
    
    2022年1月15日
    
    赞1
    
    我来啦，悟空佬![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月15日
    
    赞1
    
    看时间，周五high得晚呀![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- hncscwc
    
    2022年1月14日
    
    赞1
    
    这文章非常有深度，我都看硬了 ![[让我看看]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    哈哈哈，手动@星球的东哥
    
- 田先森😎
    
    2022年1月14日
    
    赞1
    
    通俗易通，看了好多OpenFeign分析的和悟空大佬的没法比。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    应该算是比较细的了，有些不太重要的细节没有包含进去。
    
- 阳光心情
    
    2022年1月14日
    
    赞1
    
    卧槽，又是原理，看的头疼太卷了。什么时候搞个堆外内存泄漏原理和解决方案，最近在搞这个，估计知道的人不多😎
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    你这个才是真的卷啊，又是性能优化![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- hncboy
    
    2022年1月14日
    
    赞1
    
    太强了。
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    每天进步一点点
    
- 朱晋君
    
    2022年1月14日
    
    赞1
    
    这个太肝了，一时半会看不完，先收藏
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    还有Ribbon的也收藏看下![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- fighter3
    
    2022年1月14日
    
    赞1
    
    太顶了
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    大佬换头像了呀
    
- 阿秀
    
    2022年1月14日
    
    赞1
    
    妈呀，这个太硬了![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    谢谢阿秀的打赏![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 杨乐多
    
    2022年1月14日
    
    赞1
    
    补充了好多我不够细的地方![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    可以可以，对大家有帮助就好![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- bin
    
    2022年1月14日
    
    赞1
    
    终于等到了猴哥的硬核新作，必须在看
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    感谢老铁
    
    bin
    
    2022年1月14日
    
    赞1
    
    爱猴哥![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- SnailClimb
    
    2022年1月14日
    
    赞1
    
    🐮 硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    应该算是比较细了，写这个框架的人是真的强
    
- 幻雨星辰
    
    2022年1月14日
    
    赞1
    
    辛苦了
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    是不是值个在看![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 草鱼
    
    2022年1月14日
    
    赞1
    
    无敌硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    写了三天，有点麻了
    
- 果宝小特工
    
    湖北1月22日
    
    赞
    
    首先感谢博主的分享，真的写的很棒。博主说的 第六点“最后将这个 holder 注册到 Spring 容器中”，我分析了，貌似不是把这个holder放到容器中，这个地方只是注册了BeanDefinition吧。然后容器会根据BeanDefinition来创建bean。最后在spring的bean中注册的都是FactoryBean。当依赖注入的时候会根据这个FactoryBean来创建代理对象。这个创建代理对象的时候会经行MVC解析，封装成Map源数据。不知道我的理解是否正确。最后非常感谢博主的文章。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 阮超
    
    2022年4月14日
    
    赞
    
    画图工具用的是什么？
    
    悟空聊架构
    
    作者2022年4月14日
    
    赞
    
    processon~
    
- Sinner
    
    2022年4月12日
    
    赞
    
    给大佬点个赞
    
    悟空聊架构
    
    作者2022年4月12日
    
    赞
    
    多谢大佬支持，还有ribbon的也可以看看![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    Sinner
    
    2022年4月12日
    
    赞
    
    都看了![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 磊哥
    
    2022年1月17日
    
    赞
    
    空哥，很强
    
    悟空聊架构
    
    作者2022年1月17日
    
    赞
    
    磊哥的面试系列也很强，都到14了，666
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SfAHMuUxqJ2JicQpKzblLHz64qoibVa3ATNA4rH8mIYXAF3OErAzxFKHzf5qiaiblb4rAMuAXXMJHEcKcvaHv4ia9rA/300?wx_fmt=png&wxfrom=18)

悟空聊架构

491030

46

写留言

**留言 46**

- 悟空呀
    
    2022年1月14日
    
    赞6
    
    这篇理论性很强，图画了很久，求个在看，老铁们![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    置顶
    
- 悟空呀
    
    2022年1月15日
    
    赞1
    
    补充：JDK 动态代理可以理解为动态生成了一个匿名类，然后实现了 FeignClient 接口，基于这个匿名类，创建了一个对象，也就是 proxy。对接口的调用都转交给动态代理对象处理了。
    
    置顶
    
- 一枝花算不算浪漫
    
    2022年1月14日
    
    赞2
    
    卷王之王![[色]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞3
    
    终于卷出来了，全网最细应该不过分吧![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    一枝花算不算浪漫
    
    2022年1月14日
    
    赞5
    
    不过分 这次全网都知道你最细了![[666]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 薛定谔的萌虎
    
    2022年1月14日
    
    赞2
    
    硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞3
    
    那必须的，毕竟写了这么久
    
- 麦田的守望者
    
    2022年1月14日
    
    赞2
    
    先收藏，慢慢研究
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    Ribbon 和 Eurkea 的也一起收藏下吧![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 田先森😎
    
    2022年1月15日
    
    赞1
    
    坐等大佬的Hystrix![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月15日
    
    赞1
    
    催更了![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[委屈]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- Cloud
    
    2022年1月15日
    
    赞1
    
    我来啦，悟空佬![[旺柴]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月15日
    
    赞1
    
    看时间，周五high得晚呀![[奸笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- hncscwc
    
    2022年1月14日
    
    赞1
    
    这文章非常有深度，我都看硬了 ![[让我看看]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    哈哈哈，手动@星球的东哥
    
- 田先森😎
    
    2022年1月14日
    
    赞1
    
    通俗易通，看了好多OpenFeign分析的和悟空大佬的没法比。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    应该算是比较细的了，有些不太重要的细节没有包含进去。
    
- 阳光心情
    
    2022年1月14日
    
    赞1
    
    卧槽，又是原理，看的头疼太卷了。什么时候搞个堆外内存泄漏原理和解决方案，最近在搞这个，估计知道的人不多😎
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    你这个才是真的卷啊，又是性能优化![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- hncboy
    
    2022年1月14日
    
    赞1
    
    太强了。
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    每天进步一点点
    
- 朱晋君
    
    2022年1月14日
    
    赞1
    
    这个太肝了，一时半会看不完，先收藏
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    还有Ribbon的也收藏看下![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- fighter3
    
    2022年1月14日
    
    赞1
    
    太顶了
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    大佬换头像了呀
    
- 阿秀
    
    2022年1月14日
    
    赞1
    
    妈呀，这个太硬了![[破涕为笑]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    谢谢阿秀的打赏![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[社会社会]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 杨乐多
    
    2022年1月14日
    
    赞1
    
    补充了好多我不够细的地方![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    可以可以，对大家有帮助就好![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- bin
    
    2022年1月14日
    
    赞1
    
    终于等到了猴哥的硬核新作，必须在看
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    感谢老铁
    
    bin
    
    2022年1月14日
    
    赞1
    
    爱猴哥![[爱心]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- SnailClimb
    
    2022年1月14日
    
    赞1
    
    🐮 硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    应该算是比较细了，写这个框架的人是真的强
    
- 幻雨星辰
    
    2022年1月14日
    
    赞1
    
    辛苦了
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    是不是值个在看![[机智]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 草鱼
    
    2022年1月14日
    
    赞1
    
    无敌硬核
    
    悟空聊架构
    
    作者2022年1月14日
    
    赞1
    
    写了三天，有点麻了
    
- 果宝小特工
    
    湖北1月22日
    
    赞
    
    首先感谢博主的分享，真的写的很棒。博主说的 第六点“最后将这个 holder 注册到 Spring 容器中”，我分析了，貌似不是把这个holder放到容器中，这个地方只是注册了BeanDefinition吧。然后容器会根据BeanDefinition来创建bean。最后在spring的bean中注册的都是FactoryBean。当依赖注入的时候会根据这个FactoryBean来创建代理对象。这个创建代理对象的时候会经行MVC解析，封装成Map源数据。不知道我的理解是否正确。最后非常感谢博主的文章。![[强]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 阮超
    
    2022年4月14日
    
    赞
    
    画图工具用的是什么？
    
    悟空聊架构
    
    作者2022年4月14日
    
    赞
    
    processon~
    
- Sinner
    
    2022年4月12日
    
    赞
    
    给大佬点个赞
    
    悟空聊架构
    
    作者2022年4月12日
    
    赞
    
    多谢大佬支持，还有ribbon的也可以看看![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
    Sinner
    
    2022年4月12日
    
    赞
    
    都看了![[呲牙]](https://res.wx.qq.com/mpres/zh_CN/htmledition/comm_htmledition/images/pic/common/pic_blank.gif)
    
- 磊哥
    
    2022年1月17日
    
    赞
    
    空哥，很强
    
    悟空聊架构
    
    作者2022年1月17日
    
    赞
    
    磊哥的面试系列也很强，都到14了，666
    

已无更多数据