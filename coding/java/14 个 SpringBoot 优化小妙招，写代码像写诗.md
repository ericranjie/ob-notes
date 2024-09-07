# 

点击关注 👉 顶级架构师

 _2024年09月06日 17:31_ _浙江_

**推荐关注**

![](http://mmbiz.qpic.cn/mmbiz_png/lgCBBmmGCU3e21mtriazDdaduqeb7hTzeIJ3JbK3RpQQUv1STZaN0nnTfuwfTtRe0tkQDNTVK1cKfzo212fXOsg/300?wx_fmt=png&wxfrom=19)

**Python人工智能技术**

一个有内涵的公众号。Python、机器学习、数据分析、算法、职场、一线大厂干货等知识，全力打造有趣的学习平台，一定不能错过。

公众号

顶级架构师后台回复 **1024** 有特别礼包

  

来源：Java后端编程

链接：blog.csdn.net/weixin_44912855/article/details/120866194

  

上一篇：[要不要发给你们后端？ 看看人家后端 API 接口写的，那叫一个优雅！](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247575070&idx=1&sn=82e972c297759c804f8fe915c1d28c77&chksm=e8db07bbdfac8eadfc046d970ca1541e98cd933c945a5347a2f2280aad8cede37666b887d83e&scene=21#wechat_redirect)

  

****大家好，我是顶级架构师。****

  

[**最新的 GPT-4o 在国内可以直接使用 ！**](http://mp.weixin.qq.com/s?__biz=MzkyMjY4NDgzNQ==&mid=2247483916&idx=1&sn=f6ddddf7dd27d669586ac5d988d3c236&chksm=c1f1dec0f68657d6ffae3ee1fefb2a60a90ae069e4eeed80a582b092cc71a9f129cd2221bdd4&scene=21#wechat_redirect)

  

- [1.定义配置文件信息](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [2. 用@RequiredArgsConstructor代替@Autowired](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [3.代码模块化](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [4. 抛异常而不是返回](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [5. 减少不必要的db](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [6. 不要返回null](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [7. if else](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [8. 减少controller业务代码](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [9. 利用好Idea](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [10. 阅读源码](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [11. 设计模式](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [12. 拥抱新知识](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [13. 基础问题](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    
- [14. 判断元素是否存在](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488419&idx=2&sn=0b80c7f9f73fca89b91e257a269cfada&chksm=ceabf4ebf9dc7dfdaa605a9bb92d31c9fc0a10a7a94351234181a89ba5800672c6e7da2ebfbe&scene=21#wechat_redirect)
    

  

  

到代码优化，很多人上来就是各种理论、架构、核心思路；其实优化这个事情说简单也简单，说复杂也可以很复杂，但是我觉得最重要的就是要有一个良好的编码习惯，代码"屎山”并非一朝一夕形成的，往往是经过了日积月累；因此，培养一个好的习惯，可以让我们的代码变的更加优雅、易维护，系统变的更加健壮；下面就分享14个小技巧，让优化变成顺手就完成的小事儿；

**1. 定义配置文件信息**

有时候我们为了统一管理会把一些变量放到 yml 配置文件中；而不是到处设置“魔数”，一旦那天需要修改，只需要修改配置文件即可，不需要满项目去搜索替换；

- **例如**
    
    ![Image](https://mmbiz.qpic.cn/mmbiz_png/GjuWRiaNxhnRCK3vEh7h7uS52AqXzEgmDq4iaYqebOzHjGslvzRgicue5G96h8lL1eFhxZqqHs14O3fO19kFsRiaIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1&tp=wxpic)
    
    用 @ConfigurationProperties 代替 @Value
    
- **使用方法**
    
    定义对应字段的实体
    
    ```
    @Data// 指定前缀@ConfigurationProperties(prefix = "developer")@Componentpublic class DeveloperProperty {    private String name;    private String website;    private String qq;    private String phoneNumber;}@Data// 指定前缀@ConfigurationProperties(prefix = "developer")@Componentpublic class DeveloperProperty {    private String name;    private String website;    private String qq;    private String phoneNumber;}
    ```
    
    使用时注入这个bean
    
    ```
    @RestController@RequiredArgsConstructorpublic class PropertyController {     final DeveloperProperty developerProperty;     @GetMapping("/property")    public Object index() {       return developerProperty.getName();    }}
    ```
    

## **2. 用@RequiredArgsConstructor代替@Autowired**

我们都知道注入一个 bean 有三种方式哦（**set 注入**,**构造器注入**,**注解注入**），Spring 推荐我们使用构造器的方式注入 Bean

我们来看看上段代码编译完之后的样子

![Image](https://mmbiz.qpic.cn/mmbiz_png/GjuWRiaNxhnRCK3vEh7h7uS52AqXzEgmDQUVYgiaXB4GlR4NibDXb0oWBovycVNQYacO5ZOVFKOibstyAbTz9v41HQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1&tp=wxpic)

@RequiredArgsConstructor 注解由**lombok**提供

## **3.代码模块化**

阿里巴巴 Java 开发手册中说到每个方法的代码不要超过 50 行（我没记错的话），在实际的开发中我们要善于拆分自己的接口或方法, 做到一个方法只处理一种逻辑,说不定以后某个功能就用到了, 拿来即用。

![Image](https://mmbiz.qpic.cn/mmbiz_png/GjuWRiaNxhnRCK3vEh7h7uS52AqXzEgmDbuGBLY0tUQcuqgXAdG87alPGDqDgEeyEVWhh0T2aHrSP0QibK6MT93w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1&tp=wxpic)

## **4. 抛异常而不是返回**

在写业务代码的时候，经常会根据不同的结果返回不同的信息，尽量减少返回，会显得代码比较乱

- **反例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- **正例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

## **5. 减少不必要的db**

尽可能的减少对数据库的查询

**举例子**

删除一个服务（已下架或未上架的才能删除），之前有看别人写的代码，会先根据id查询该记录，然后做一些判断

- **反例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- **正例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

## **6. 不要返回 null**

避免调用方法时，造成不必要的空指针

- **反例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- **正例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

## **7. if else**

不要太多了if else if，可以试试策略模式代替

## **8. 减少controller业务代码**

业务代码尽量放到service层进行处理，后期维护起来也好操作而且美观

- **反例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    
- **正例**
    
    ![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)
    

## **9. 利用好IDEA**

目前为止市面上的企业基本都用idea作为开发工具了吧

**举一个小例子**

IDEA会对我们的代码进行判断，提出合理的建议

**例如：**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

它推荐我们用lanbda的形式代替，点击replace

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

## **10. 阅读源码**

一定要养成阅读源码的好习惯包括优秀的开源项目GitHub上stars:>1000, 会从中学好好多知识包括其对代码的设计思想以及高级API，面试加分（好多面试官习惯问源码相关的知识）

微信搜索公众号：架构师指南，回复：架构师 领取资料 。

## **11. 设计模式**

23种设计模式，要尝试代码中运用设计模式思想，写出的代码即规范又美观还高大上哈哈。

## **12. 拥抱新知识**

像我们这种工作年限少的程序员，我觉得要多学习自己认知之外的知识，不能每天crud，有机会就多用用有点难度的知识，没有机会（项目较传统），可以自己下班多些相关demo练习

## **13. 基础问题**

- **Map遍历**
    
    ```
    HashMap<String, String> map = new HashMap<>();map.put("name", "du");for (String key : map.keySet()) {   String value = map.get(key);} map.forEach((k, v) -> {}); // 推荐for (Map.Entry<String, String> entry : map.entrySet()) { }
    ```
    
- **optional 判空**
    
    ```
    //获取子目录列表public List<CatalogueTreeNode> getChild(String pid) {    if (V.isEmpty(pid)) {        pid = BasicDic.TEMPORARY_DIRECTORY_ROOT;    }    CatalogueTreeNode node = treeNodeMap.get(pid);     return Optional.ofNullable(node)              .map(CatalogueTreeNode::getChild)              .orElse(Collections.emptyList());}
    ```
    
- **递归**
    
    大数据量的递归时，避免在递归方法里new对象，可以试试把对象当作方法参数进行传递使用
    
- **注释**
    
    类 接口方法 注解 较复杂的方法 注释都要写而且要写清楚, 有时候写注释不是给别人看的 而是给自己看的
    

## **14. 判断元素是否存在**

hashSet 而不是 list，list 判断一个元素是否存在的代码

```
ArrayList<String> list = new ArrayList<>(); // 判断a是否在list中 for (int i = 0; i < list.size(); i++)       if ("a".equals(elementData[i]))          return i;
```

由此可见其复杂度为On，而hashSet底层采用hashMap作为数据结构进行存储，元素都放到map的key（即链表中）

```
HashSet<String> set = new HashSet<>(); // 判断a是否在set中 int index = hash(a); return getNode(index) != null
```

由此可见其复杂度为O1。

**欢迎大家进行观点的探讨和碰撞，各抒己见。如果你有疑问，也可以找我沟通和交流。**扩展：[接私活儿](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247534656&idx=2&sn=2781baec773a9340091436c521430648&chksm=e8dae9e5dfad60f33204393e3c333800e8fb8dd39fd43064d496b665cd31c0a61b742f2f23b2&scene=21#wechat_redirect)[](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247534656&idx=2&sn=2781baec773a9340091436c521430648&chksm=e8dae9e5dfad60f33204393e3c333800e8fb8dd39fd43064d496b665cd31c0a61b742f2f23b2&scene=21#wechat_redirect)

  

[上周，又劝退十几个了。。。](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247570612&idx=2&sn=61b7b724f86908509f208ddb30485f2b&chksm=e8db7511dfacfc07ec8279b81ee0f3a694d7750d2c4f8057c20b7e265a93bcc7fd62aba86ded&scene=21#wechat_redirect)

[ChatGPT 4.0 国内直接用 ！！！](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247571156&idx=2&sn=5bfca7ebe0cc8cefa51739c1cdba1194&chksm=e8db7771dfacfe670c7dd2df0f1169bdd04ef0a15fe3848edf7ff47162300016ba72859b986e&scene=21#wechat_redirect)

最后给大家推荐一个ChatGPT 4.0国内网站，是我们团队一直在使用的，我们对接是OpenAI官网的账号，给大家打造了一个一模一样ChatGPT，很多粉丝朋友现在也都通过我拿这种号，价格不贵，关键还有售后。

一句话说明：用官方一半价格的钱，一句话说明:用跟官方 ChatGPT4.0 一模一样功能，无需魔法，无视封号，不必担心次数不够。

最大优势：可实现会话隔离！突破限制：官方限制每个账号三小时可使用40次4.0本网站可实现次数上限之后，手动切换下一个未使用的账号【相当于一个4.0帐号，同享受一百个账号轮换使用权限】

[![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MzkwODI0Njc1OA==&mid=2247496302&idx=2&sn=075c6581c8a4f4229db1752d75d8689b&chksm=c0ce59e7f7b9d0f102b08845199c766a425df62660d11fa965a27b8c407e13bb52899a721446&scene=21#wechat_redirect)

为了跟上AI时代我干了一件事儿，我创建了一个知识星球社群：ChartGPT与副业。想带着大家一起探索**ChatGPT和新的AI时代**。

有很多小伙伴搞不定ChatGPT账号，于是我们决定，凡是这三天之内加入ChatPGT的小伙伴，我们直接送一个正常可用的永久ChatGPT独立账户。

不光是增长速度最快，我们的星球品质也绝对经得起考验，短短一个月时间，我们的课程团队发布了**8个专栏、18个副业项目**：

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**简单说下这个星球能给大家提供什么：**

1、不断分享如何使用ChatGPT来完成各种任务，让你更高效地使用ChatGPT，以及副业思考、变现思路、创业案例、落地案例分享。

2、分享ChatGPT的使用方法、最新资讯、商业价值。

3、探讨未来关于ChatGPT的机遇，共同成长。

4、帮助大家解决ChatGPT遇到的问题。

5、**提供一整年的售后服务，一起搞副业**

星球福利：

1、加入星球4天后，就送ChatGPT独立账号。

2、邀请你加入ChatGPT会员交流群。

3、赠送一份完整的ChatGPT手册和66个ChatGPT副业赚钱手册。

其它福利还在筹划中... **不过，我给你大家保证，加入星球后，收获的价值会远远大于今天加入的门票费用 ！**

本星球第一期原价**399**，目前属于试运营，早鸟价**169**，每超过50人涨价10元，星球马上要来一波大的涨价，如果你还在犹豫，可能最后就要以**更高价格加入了**。。

**早就是优势。建议大家尽早以便宜的价格加入！**

#### 

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

最后给读者整理了一份**BAT**大厂面试真题，需要的可扫码回复“**面试题**”即可获取。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

****![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)****  

公众号后台回复 **架构** 或者 **架构整洁** 有惊喜礼包！

**顶级架构师交流群**

 **「顶级架构师」建立了读者架构师交流群，大家可以添加小编微信进行加群。欢迎有想法、乐于分享的朋友们一起交流学习。**

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

扫描添加好友邀你进架构师群，加我时注明**【****姓名+公司+职位】**  

  

**版权申明：内容来源网络，版权归原作者所有。如有侵权烦请告知，我们会立即删除并表示歉意。谢谢。**

猜你还想看

[推荐一套开源通用后台管理系统（附源码）](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247517463&idx=1&sn=39841b9fb02e185d4cb06df7f5b7cc02&chksm=e8da24b2dfadada4eea431a3df5ea184203cdc1cf4f833549f80956328901c5e005f8d388583&scene=21#wechat_redirect)

[牛逼啊！接私活必备的 N 个开源项目！赶快收藏吧（附源码合集第九期）！](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247565796&idx=2&sn=ff266d1782b8147c3c809a10f45af012&chksm=e8db6041dface957e9cb0ea634fda9e977d1aeaf598d54f8475cdd68744a4ce219d1d0e72c81&scene=21#wechat_redirect)

[看看人家那 IM 即时通讯系统，那叫一个优雅（附源码）](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247520498&idx=1&sn=247bed1ab2e30cad54981a3eeff00810&chksm=e8da3157dfadb84118310e1067c0af1e8fc3e41b15d2dfde8ab61889b8522f603bbda08bed8c&scene=21#wechat_redirect)  

[一张图理解微服务架构设计](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247571243&idx=1&sn=85fe76423dda1d91edc3ead0a7bfa005&chksm=e8db768edfacff98ef1013523aec35fad53a8e92013d0dfe6935f651bfb5009c86a7ae7a4117&scene=21#wechat_redirect)  

[面试必问：Redis 如何实现库存扣减操作？](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247571156&idx=3&sn=fc3042d3fc2eea42b8faec7b122cda28&chksm=e8db7771dfacfe6732c51ac9dd3690e081c9a9baa3d157b64a86a3f9affcbd354f4c13e73a37&scene=21#wechat_redirect)  

[10条sql语句优化的建议](http://mp.weixin.qq.com/s?__biz=MzIzNjM3MDEyMg==&mid=2247571961&idx=1&sn=738671233a4192c132aff85d4bbafe79&chksm=e8db785cdfacf14acf6999c9bd8f3c70a5aa566ae74bf2a0eeaea38f022e9b28c0ea6ad0d117&scene=21#wechat_redirect)

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

最近面试BAT，整理一份面试资料《**Java面试BAT通关手册**》，覆盖了Java核心技术、JVM、Java并发、SSM、微服务、数据库、数据结构等等。

获取方式：点“在看”，关注公众号并回复 手册 领取，更多内容陆续奉上。

**明天见(｡･ω･｡)**

Read more

Reads 222

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/sz_mmbiz_png/gHvX5TiczgWlCsOPBib3qa34WKGOy72FcvqTvt9icWjB0223JqDtJtD25EmBcaFxlJJ8P2r6KEADI3KYw7H1zuMRg/300?wx_fmt=png&wxfrom=18)

顶级架构师

1261

Comment

Comment

**Comment**

暂无留言