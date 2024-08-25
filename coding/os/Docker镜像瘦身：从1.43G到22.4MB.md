# 

Linux公社

 _2021年10月10日 08:20_

点击上方蓝字 ● 关注Linux公社 

Docker 镜像的大小对于系统的 CI/CD 等都有影响，尤其是云部署场景。我们在生产实践中都会做瘦身的操作，尽最大的可能使用 Size 小的镜像完成功能。

下文是一个简单的 ReactJS 程序上线的瘦身体验，希望可以帮助大家找到镜像瘦身的方向和灵感。

如果你正在做 Web 开发相关工作，那么你可能已经知道容器化的概念，以及知道它强大的功能等等。

但在使用 Docker 时，镜像大小至关重要。我们从 create-react-app （https://reactjs.org/docs/create-a-new-react-app.html）获得的样板项目通常都超过 1.43 GB。

今天，我们将容器化一个 ReactJS 应用程序，并学习一些关于如何减少镜像大小并提高性能的技巧。

我们将以 ReactJS 为例，但它适用于任何类型的 NodeJS 应用程序。

### 步骤 1：创建项目

①借助脚手架通过命令行模式创建 React 项目：

```
npx create-react-app docker-image-test
```

②命令执行成功后将生成一个基础 React 应用程序架构。

③我们可以进入项目目录安装依赖并运行项目：

```
cd docker-image-testyarn installyarn start
```

④通过访问 http://localhost:3000 可以访问已经启动的应用程序。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 步骤 2：构建第一个镜像

①在项目的根目录中创建一个名为 Dockerfile 的文件，并粘贴以下代码：

```
FROM node:12WORKDIR /appCOPY package.json ./RUN yarn installCOPY . .EXPOSE 3000CMD ["yarn", "start"]
```

②注意，这里我们从 Docker 仓库获得基础镜像 Node:12，然后安装依赖项并运行基本命令。

③现在可以通过终端为容器构建镜像：

```
docker build -t docker-image-test .
```

④Docker 构建镜像完成之后，你可以使用此命令查看已经构建的镜像：

```
docker images
```

在查询结果列表的顶部，是我们新创建的图像，在最右边，我们可以看到图像的大小。目前是 1.43GB。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

⑤我们使用以下命令运行镜像：

```
docker run --rm -it -p 3000:3000/tcp docker-image-test:latest
```

打开浏览器并且刷新页面验证其可以正常运行。

### 步骤 3：修改基础镜像

①先前的配置中我们用 node:12 作为基础镜像。但是传统的 Node 镜像是基于 Ubuntu 的，对于我们简单的 React 应用程序来说这大可不必。

②从 DockerHub（官方 Docker 镜像注册表）中我们可以看到，基于 alpine-based 的 Node 镜像比基于 Ubuntu 的镜像小得多，而且它们的依赖程度非常低。

③下面显示了这些基本图像的大小比较：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在我们将使用node:12-alpine作为我们的基础镜像，看看会发生什么。

```
FROM node:12-alpineWORKDIR /appCOPY package.json ./RUN yarn installCOPY . .EXPOSE 3000CMD ["yarn", "start"]
```

然后我们以此构建我们的镜像，并与之前做对比。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

哇！我们的镜像大小减少到只有 580MB，这是一个很大的进步。但还能做得更好吗？

### 步骤 4：多级构建

①在之前的配置中，我们会将所有源代码也复制到工作目录中。

②但这大可不必，因为从发布和运行来看我们只需要构建好的运行目录即可。因此，现在我们将引入多级构建的概念，以减少不必要的代码和依赖于我们的最终镜像。

③配置是这样的：

```
# STAGE 1FROM node:12-alpine AS buildWORKDIR /appCOPY package.json ./RUN yarn  installCOPY . /appRUN yarn build# STAGE 2FROM node:12-alpineWORKDIR /appRUN npm install -g webserver.localCOPY --from=build /app/build ./buildEXPOSE 3000CMD webserver.local -d ./build
```

④在第一阶段，安装依赖项并构建我们的项目。

⑤在第二阶段，我们复制上一阶段构建产物目录，并使用它来运行应用程序。

⑥这样我们在最终的镜像中就不会有不必要的依赖和代码。

接下来，构建镜像成功后并从列表中查看镜像：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在我们的镜像大小只有 97.5MB。这简直太棒了。

### 步骤 5：使用 Nginx

①我们正在使用 Node 服务器运行 ReactJS 应用程序的静态资源，但这不是静态资源运行的最佳选择。

②我们尝试使用 Nginx 这类更高效、更轻量级的服务器来运行资源应用程序，也可以尽可能提高其性能，并且减少镜像的量。

③我们最终的 Docker 配置文件看起来像这样：

```
# STAGE 1FROM node:12-alpine AS buildWORKDIR /appCOPY package.json ./RUN yarn  installCOPY . /appRUN yarn build# STAGE 2FROM nginx:stable-alpineCOPY --from=build /app/build /usr/share/nginx/htmlEXPOSE 80CMD ["nginx", "-g", "daemon off;"]
```

④我们正在改变 Docker 配置的第二阶段，以使用 Nginx 来服务我们的应用程序。

⑤然后使用当前配置构建镜像。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

⑥镜像大小减少到只有 22.4MB！

⑦同时，我们正在使用一个性能更好的服务器来服务我们出色的应用程序。

⑧我们可以使用以下命令验证应用程序是否仍在工作。

```
docker run --rm  -it -p 3000:80/tcp docker-image-test:latest
```

⑨注意，我们将容器的 80 端口暴露给外部，因为默认情况下，Nginx 将在容器内部的 80 端口上可用。

所以这些是一些简单的技巧，你可以应用到你的任何 NodeJS 项目，以大幅减少镜像大小。

现在，您的容器确实更加便携和高效了。今天就到这里。编码快乐！

> 作者：张亚龙译 
> 
> 出处：转载自公众号分布式实验室（ID：dockerone）

> **关注我们  
> **
> 
> 长按或扫描下面的二维码关注Linux公社
> 
>   
> 
> ![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
> 
> ****关注**Linux公社，添加“****星标****”**
> 
> **每天**获取**技术干货，让我们一起成长**
> 
> **合作联系：root@linuxidc.net**

Docker27

Docker · 目录

上一篇5分钟教程：如何在 Docker 容器中运行 Nginx下一篇Docker + IntelliJ IDEA，提升10倍生产力！

阅读 3709

​

写留言

**留言 1**

- 軍爺
    
    2021年10月12日
    
    赞1
    
    构建好直接丢在ng下面不好么 折腾啥啊
    

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/jhtEbpg4m6G84lgroY9BgpU8R6RPj81icLSHhygstwMxGhjduVoic642W3AgNstLf3GQcvZYx9zeibDQrfCS6hEibw/300?wx_fmt=png&wxfrom=18)

Linux公社

3分享4

1

写留言

**留言 1**

- 軍爺
    
    2021年10月12日
    
    赞1
    
    构建好直接丢在ng下面不好么 折腾啥啊
    

已无更多数据