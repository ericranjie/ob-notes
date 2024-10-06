架构师社区
 _2021年09月09日 11:30_
**来自：Nginx-GUI入门**
**链接：https://leanote.zzzmh.cn/blog/post/5cc7f63616199b068300001c**

### 需求

**nginx 可视化管理，例如**

- 配置管理
    
- 性能监控
    
- 日志监控
    
- 其他配置
    

### 方案

目前已实现前两条:配置管理，和性能监控  
日志分析监控这块还需要另找方案实现！

目前方案直接套用github大神开发的nginx-gui

github地址：https://github.com/onlyGuo/nginx-gui

这个东西真的要吹一波，太好用了

而且源码公开，解决了我这种java出身的linux菜鸟的一大难题！

界面截图：  

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk56krM8DD1RAH3dicx5KQhU1sia3ic6ThzIsv5MMjiadCUNfM02iaVUqXnm6oTeYjia9p1hJEomu1HDvmtdQ/640?wx_fmt=png&wxfrom=13&tp=wxpic)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 说明

先说明下，我也是刚才现学的，只是写下折腾的过程和碰到的问题

详细的用法之类的还是建议访问作者的github和作者的博客查看

作者github：https://github.com/onlyGuo/nginx-gui

作者博客：http://bl.321aiyi.com/2019/03/18/nginx-gui/

### 折腾

**一 下载和配置**

首先到作者github说明页面，下载对应系统版本的安装包

需要注意的是linux版本有一段描述不可忽视

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

配置步骤如下：

#### 1 下载并解压 `Nginx-GUI-For-Linux-1.0.zip`

略

#### 2 修改配置文件

文件位置：`conf/conf.properties`

```
# nginx 安装路径nginx.path = /usr/local/Cellar/nginx/1.15.12# nginx 配置文件全路径nginx.config = /Users/gsk/dev/apps/nginx-1.15.12/conf/nginx.conf# account.admin = admin
```

#### 3 重命名(此步骤仅linux版本需要)

根据原作者的描述  
针对linux 64位版本  
需要将 `lib/bin/`  
下的 `java_vms` 文件  
重命名为 `java_vms_nginx_gui`

**二 在服务器上运行**

前面的步骤都完成以后，直接打包发布到服务器

```
# 赋权sudo chmod -R 777 nginx-gui/# 后台启动nohup bash /root/web/nginx-gui/startup.sh > logs/nginx-gui.out &
```

访问默认端口 8889 默认账号密码都是admin

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 遗留问题

目前实现的有

- 性能监控
    
- 可视化配置
    

未能实现的是

- 日志分析
    
- 访问统计
    

阅读 1.5万

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk55KKLFaGCDRURMvFtPXf9fZXJOHOFsA3Ye8Qbibf3qHLkBQNpdjicAVpPf2T03EcakjAFbwqicjXSibXA/300?wx_fmt=png&wxfrom=18)

架构师社区

54132

写留言

写留言

**留言**

暂无留言