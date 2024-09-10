
ImportNew

 _2022年01月16日 17:51_

（给ImportNew加星标，提高Java技能）

本文介绍策略模式的具体应用以及Map+函数式接口如何 “更完美” 的解决 if-else的问题。

### 文章目录

- 需求
    
- 策略模式
    
- Map+函数式接口
    
- 最后捋一捋本文讲了什么
    

### 需求

最近写了一个服务：根据优惠券的类型resourceType和编码resourceId来 查询 发放方式grantType和领取规则

##### 实现方式：

- 根据优惠券类型resourceType -> 确定查询哪个数据表
    
- 根据编码resourceId -> 到对应的数据表里边查询优惠券的派发方式grantType和领取规则
    

优惠券有多种类型，分别对应了不同的数据库表：

- 红包 —— 红包发放规则表
    
- 购物券 —— 购物券表
    
- QQ会员
    
- 外卖会员
    

实际的优惠券远不止这些，这个需求是要我们写一个业务分派的逻辑

第一个能想到的思路就是if-else或者switch case：

```c
switch(resourceType){ case "红包":   查询红包的派发方式   break; case "购物券":   查询购物券的派发方式  break; case "QQ会员" :  break; case "外卖会员" :  break; ...... default : logger.info("查找不到该优惠券类型resourceType以及对应的派发方式");  break;}
```

如果要这么写的话， 一个方法的代码可就太长了，影响了可读性。（别看着上面case里面只有一句话，但实际情况是有很多行的）

而且由于 整个 if-else的代码有很多行，也不方便修改，可维护性低。

### 策略模式

策略模式是把 if语句里面的逻辑抽出来写成一个类，如果要修改某个逻辑的话，仅修改一个具体的实现类的逻辑即可，可维护性会好不少。

以下是策略模式的具体结构
![[Pasted image 20240910195155.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

策略模式在业务逻辑分派的时候还是if-else，只是说比第一种思路的if-else 更好维护一点。

```c
switch(resourceType){ case "红包":   String grantType=new Context(new RedPaper()).ContextInterface();  break; case "购物券":   String grantType=new Context(new Shopping()).ContextInterface();  break;  ...... default : logger.info("查找不到该优惠券类型resourceType以及对应的派发方式");  break;
```

但缺点也明显：

- 如果 if-else的判断情况很多，那么对应的具体策略实现类也会很多，上边的具体的策略实现类还只是2个，查询红包发放方式写在类RedPaper里边，购物券写在另一个类Shopping里边；那资源类型多个QQ会员和外卖会员，不就得再多写两个类？有点麻烦了
    
- 没法俯视整个分派的业务逻辑
    

### Map+函数式接口

用上了Java8的新特性lambda表达式

- 判断条件放在key中
    
- 对应的业务逻辑放在value中
    

这样子写的好处是非常直观，能直接看到判断条件对应的业务逻辑

> 需求：根据优惠券(资源)类型resourceType和编码resourceId查询派发方式grantType

上代码：

```c
@Servicepublic class QueryGrantTypeService {     @Autowired    private GrantTypeSerive grantTypeSerive;    private Map<String, Function<String,String>> grantTypeMap=new HashMap<>();    /**     *  初始化业务分派逻辑,代替了if-else部分     *  key: 优惠券类型     *  value: lambda表达式,最终会获得该优惠券的发放方式     */    @PostConstruct    public void dispatcherInit(){        grantTypeMap.put("红包",resourceId->grantTypeSerive.redPaper(resourceId));        grantTypeMap.put("购物券",resourceId->grantTypeSerive.shopping(resourceId));        grantTypeMap.put("qq会员",resourceId->grantTypeSerive.QQVip(resourceId));    }     public String getResult(String resourceType){        //Controller根据 优惠券类型resourceType、编码resourceId 去查询 发放方式grantType
Function<String,String> result=getGrantTypeMap.get(resourceType);        if(result!=null){         //传入resourceId 执行这段表达式获得String型的grantType
return result.apply(resourceId);        }        return "查询不到该优惠券的发放方式";    }}
```

如果单个 if 语句块的业务逻辑有很多行的话，我们可以把这些 业务操作抽出来，写成一个单独的Service，即：

```java
//具体的逻辑操作
@Servicepublic class GrantTypeSerive {    public String redPaper(String resourceId){        //红包的发放方式
return "每周末9点发放";
}								  public String shopping(String resourceId){        //购物券的发放方式
return "每周三9点发放";
}    public String QQVip(String resourceId){        //qq会员的发放方式
return "每周一0点开始秒杀";    }}
```

入参String resourceId是用来查数据库的，这里简化了，传参之后不做处理。

用http调用的结果：

```java
@RestControllerpublic class GrantTypeController {    @Autowired    private QueryGrantTypeService queryGrantTypeService;    @PostMapping("/grantType")    public String test(String resourceName){        return queryGrantTypeService.getResult(resourceName);    }}
```
![[Pasted image 20240910195313.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

用Map+函数式接口也有弊端：

- 你的队友得会lambda表达式才行啊，他不会让他自己百度去
    

### 最后捋一捋本文讲了什么

策略模式通过接口、实现类、逻辑分派来完成，把 if语句块的逻辑抽出来写成一个类，更好维护。

Map+函数式接口通过Map.get(key)来代替 if-else的业务分派，能够避免策略模式带来的类增多、难以俯视整个业务逻辑的问题。

> 转自：https://blog.csdn.net/qq_44384533/article/details/109197926

  

- EOF -

推荐阅读  点击标题可跳转

[精妙绝伦的并发艺术品 — ConcurrentHashMap是如何保证线程安全的](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651507050&idx=2&sn=c3e4fb7916091fe780abd25c20e0833d&chksm=bd25a5158a522c0336dd036b96de6b60c2ea6c90702f2abb9567ac5dfbfafecca4a141876919&scene=21#wechat_redirect)  

[一个 HashMap 跟面试官扯了半个小时](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651503297&idx=2&sn=5f19e03d6b86789b3b28af5cee97d1e9&chksm=bd25d6be8a525fa8b8d16745d3454ddc4aa562571964d88eb9fd4e2cb274f8a9ae806bf0b3d5&scene=21#wechat_redirect)

[阿里二面：main 方法可以继承吗？](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651508534&idx=1&sn=dbe47cc505ca306c80db884fae973b46&chksm=bd25a3498a522a5fb4c338276bfd8af8699618e0e93c76aea2a11090cd7968e4ff07074a9c93&scene=21#wechat_redirect)

  

  

看完本文有收获？请转发分享给更多人  

**关注「ImportNew」，提升Java技能**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点赞和在看就是最大的支持❤️

阅读 1.0万

​