

[](https://juejin.cn/user/1204720475840791/posts)

2024-04-0460,670阅读11分钟

## 引言

在传统的Java应用程序开发和部署场景中，开发者往往需要经历一系列复杂的步骤才能将应用成功部署到生产环境。例如，对于基于Servlet规范的Java Web应用，开发完成后通常会被打包成WAR格式，然后部署到像Apache Tomcat、Jetty这样的Web容器中。这一过程中，不仅要管理应用本身的编译产物，还需要处理各种第三方依赖库的版本和加载顺序，同时在服务器端进行相应的配置以确保应用正常运行。

随着Spring Boot产生，它以其开箱即用、约定优于配置的理念彻底改变了Java应用的开发体验。其中一个标志性特征便是Spring Boot应用可以被打包成一个可直接运行的jar文件，无需外部容器的支持。

当提及“Spring Boot的jar可以直接运行”，我们不禁好奇：这背后究竟是怎样的机制让一个简单的命令行操作就能启动一个完整的Web服务或任何类型的Java应用呢？本文将深入剖析Spring Boot的打包过程和运行原理，揭示其jar包是如何巧妙地集成了依赖、嵌入了Web容器、实现了自动配置等功能，从而使得开发人员能够迅速地将应用部署到任何支持Java的环境中。

![springboot的jar包为什么可以直接运行.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/791c04b74cba4920826e92066fd9fdcf~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1952&h=982&s=181704&e=png&b=fefafa "springboot的jar包为什么可以直接运行.png")

## SpringBoot JAR包基础概念

Fat JAR（也称作Uber JAR，也被戏称为胖Jar）是一种特殊的Java归档（JAR）文件，它将应用程序所需的全部依赖库与应用程序自身的类文件合并到了同一个JAR文件中。在Spring Boot上下文中，Fat JAR被用于构建一种完全自包含且可独立运行的应用程序包。这样的jar文件不仅仅包含项目的主代码，还包括了所有必要的第三方库、资源文件等一切运行时所需要的组件。

Fat JAR的核心特点是“自包含”，意味着只需分发这一个文件即可部署应用，无需再额外处理众多的依赖库。这种形式极大地方便了应用的快速部署与迁移，尤其适合于云端部署或者无网络环境下的安装。

而对于普通jar包来说，它通常仅包含一个模块或应用程序的一部分，主要用来封装和组织Java类及相关资源。在Java生态系统中，一个普通的jar包可能仅是一个库，或者一组相关功能的集合，但它不会包含其他依赖的jar包，因此在运行时需要与之相关的其他库一起存在于类路径中。

相比之下，Fat JAR则解决了依赖管理的问题，通过将所有的依赖都纳入其中，避免了由于类路径设置不正确导致的“缺失类”或“找不到类”的问题。在Spring Boot项目中，通过Maven或Gradle插件可以轻易地构建出这样的Fat JAR，使得最终生成的jar文件成为一个真正的“一站式”解决方案，只需使用`java -jar`命令就可以启动整个应用程序，无需预先配置复杂的类路径环境。

## Spring Boot应用打包机制

Spring Boot应用打包机制充分利用了Maven或Gradle构建工具的强大功能，旨在简化传统Java应用的构建与部署流程。其核心在于创建一个可执行的Fat JAR，使得开发者能够轻松地将整个Spring Boot应用及其依赖项打包成单个文件，从而实现一键启动和便捷部署。

我们以Maven打包为例：

对于使用Maven构建的Spring Boot应用，`spring-boot-maven-plugin`是关键插件，负责处理Fat JAR的构建。在pom.xml文件中，通常会看到如下配置：

xml

代码解读

复制代码

`<build>     <plugins>         <plugin>             <groupId>org.springframework.boot</groupId>             <artifactId>spring-boot-maven-plugin</artifactId>             <version>${spring-boot.version}</version>             <configuration>                 <!-- 可选配置项，如mainClass属性指定入口类 -->                 <mainClass>${start-class}</mainClass>             </configuration>             <executions>                 <execution>                     <goals>                         <goal>repackage</goal>                     </goals>                 </execution>             </executions>         </plugin>     </plugins> </build>`

通过`mvn package`命令，Maven首先会按照标准流程构建项目，随后`spring-boot-maven-plugin`会执行`repackage`目标，该目标会重新包装已生成的标准JAR文件，将其转换为包含所有依赖项和适当的启动器信息的Fat JAR。这样生成的JAR可以直接通过`java -jar`命令启动。

Spring Boot应用打包机制均确保了生成的包不仅包含了项目本身的类，还包含了运行时所必需的所有依赖库，以及一些特定的元数据（如MANIFEST.MF中的启动类信息）。这一特性大大简化了部署过程，并有助于提升应用的可移植性和维护性。Fat jar中的内容：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05965939c6b54adc81d5ce81cd1b7cd7~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=2154&h=672&s=174336&e=png&b=2d2c2b "image.png")

- `META-INF/`: 包含MANIFEST.MF文件和其他元数据信息，其中Main-Class属性指向Spring Boot的启动类加载器。
- `BOOT-INF/classes/`: 存放项目自身的类文件和资源文件。
- `BOOT-INF/lib/`: 放置所有依赖的jar包，包括Spring Boot starter依赖以及其他第三方库。（如果项目中有静态资源文件，也会在BOOT-INF下有对应的static、templates等目录）

## Spring Boot启动器与Loader机制

Spring Boot应用的jar包可以直接运行主要依赖于它的启动器以及Loader机制，而对于Loader机制主要利用MANIFEST.MF文件以及其内部类加载逻辑。

### MANIFEST.MF文件是什么？

`MANIFEST.MF`是`JAR`文件内的一个标准元数据文件，它包含了关于JAR包的基本信息和运行指令。在Spring Boot应用的jar包中，`MANIFEST.MF`尤为重要，因为它设置了`Main-Class`属性，指示了用于启动整个应用程序的类，这个类通常是`org.springframework.boot.loader.JarLauncher`或其他由Spring Boot提供的启动器类。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09324630083248d89cc237ffd4656654~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1528&h=604&s=121050&e=png&b=333b44 "image.png")

image.png

`Main-Class`属性指向的`JarLauncher`类是Spring Boot自定义的类加载器体系的一部分。`JarLauncher`继承自`org.springframework.boot.loader.Launcher`，专门用于启动以`Fat JAR`形式发布的Spring Boot应用。`JarLauncher`负责创建一个类加载器`LaunchedURLClassLoader`。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68f66732638b4bdc9e56b8e5ce7fbd72~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1826&h=424&s=107873&e=png&b=262626 "image.png")

image.png

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd8b33a1e8e14a62a4788846026d21fa~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1866&h=712&s=161698&e=png&b=262626 "image.png")

当通过`java -jar`命令执行Spring Boot jar包时，JVM会依据`MANIFEST.MF`中的`Main-Class`启动指定的启动器。

![JarLauncher获取MainClass源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da987259d29140438204188b00779f3f~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1754&h=522&s=109891&e=png&b=262626 "JarLauncher获取MainClass源码.png")

JarLauncher获取MainClass源码.png

Spring Boot的启动器类加载器`LaunchedURLClassLoader`首先会读取MANIFEST.MF中的附加属性，如`Start-Class`（标识应用的实际主类）和`Spring-Boot-Lib`（指向内部依赖库的位置）。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d4373cb46aa4c728058319b555ad30b~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1420&h=378&s=75442&e=png&b=272727 "image.png")

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f661fd6849ef48bd940b9a2f9b9868f2~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.avis#?w=1422&h=566&s=120872&e=png&b=333b44 "image.png")

启动类加载器工作流程如下：

1. 当启动器类加载器启动时，它会根据MANIFEST.MF中的信息来组织类路径，保证所有内部的依赖库都能正确地被加载。
2. 加载器会区分出 `BOOT-INF/classes`中的应用程序类和 `BOOT-INF/lib` 下的依赖库，分别处理并加入到类加载器的搜索路径中。
3. 加载器加载并执行实际的`Start-Class`，即应用的主类，触发Spring Boot框架的初始化和应用的启动流程。比如示例中的应用主类：`com.springboot.base.SpringBootBaseApplication`

Spring Boot的启动器和加载器机制有效地实现了对自包含jar包的管理和执行，我们无需关心复杂的类路径配置和依赖加载，只需通过一个简单的命令即可启动一个完整、独立运行的应用程序。

## 内嵌Web容器

Spring Boot的一大特色就是能够无缝整合并内嵌多种轻量级Web容器，比如：`Apache Tomcat`、`Jetty`、`Undertow`以及`Reactor Netty`（对于响应式编程模型）。内嵌Web容器的引入极大地简化了Web应用的部署流程，我们不再需要在本地或服务器上独立安装和配置Web服务器（比如以前还要在本地安装tomcat）。

当Spring Boot应用引入了`spring-boot-starter-web`依赖时，默认情况下会自动配置并启动一个内嵌的Web容器。在Spring Boot启动的过程中，内嵌容器作为应用的一部分被初始化并绑定到特定端口上，以便对外提供HTTP服务。

Spring Boot内嵌web容器的优点在于简化部署，通过将Web容器内置于应用中，只需分发单一的JAR文件，就能在干净的环境中运行应用，避免了与现有Web服务器版本冲突或配置不当等问题；同时加快了启动速度，尤其在开发和测试阶段，实现近乎即时的热重启；提高了应用的稳定性，因为开发环境和生产环境使用相同的Web容器，降低了因环境差异导致的问题；此外，虽然容器是内嵌的，但仍然可以进行全面的配置调整，如端口、连接数、SSL设置等，以满足不同场景的需求。通过内嵌Web容器，Spring Boot真正实现了“开箱即用”的理念。

## 自动配置与类路径扫描

Spring Boot的核心特性之一就是其强大的自动配置能力，它允许应用在几乎零配置的情况下快速启动并运行。

当应用启动时，Spring Boot会读取`resource/META-INF/spring.factories`文件，该文件列出了所有可用的自动配置类。当它检测到应用环境中对应的自动配置类就会生效，通过`@Configuration`注解的类创建并注册Bean到Spring容器中，从而实现Bean的自动装配。

> 这里说明下，在springboot3.x以后，就不在从resource/META-INF/spring.factories读取自动配置类了，而是从org.springframework.boot.autoconfigure.AutoConfiguration.imports中读取，这一点请参考文章：[华为二面：SpringBoot如何自定义Starter？](https://link.juejin.cn/?target=https%3A%2F%2Fwww.coderacademy.online%2Farticle%2Fspringbootcustomstarter.html "https://www.coderacademy.online/article/springbootcustomstarter.html")

并且Spring Boot还采用条件注解（如`@ConditionalOnClass`、`@ConditionalOnMissingBean`等）来智能判断何时应用特定的配置。这些注解可以根据类路径中是否存在特定类、系统属性或环境变量的值等因素，决定是否应该激活某个自动配置类。这意味着只有当满足特定条件时，相应的Bean才会被创建和注入。

而对于应用主类则是用`@SpringBootApplication`注解标识。`@SpringBootApplication`是一个复合注解，包含了`@SpringBootConfiguration`、`@EnableAutoConfiguration`和`@ComponentScan`三个注解的功能。其中

- `@SpringBootConfiguration`是一个Spring配置类，可以替代`@Configuration`注解，声明当前类是Spring配置类，里面包含了一系列`@Bean`方法或`@ConfigurationProperties`等配置。
- `@EnableAutoConfiguration`启用自动配置特性，告诉Spring Boot根据应用类路径中的依赖来自动配置Bean。Spring Boot会根据类路径扫描的结果，智能地决定哪些自动配置类应当生效。
- `@ComponentScan`会自动扫描和管理Spring组件，包括@Service、@Repository、@Controller和@Component等注解标注的类。通过该注解，Spring Boot能自动发现和管理应用中的各个组件，并将其注册为Spring容器中的Bean。

通过上述机制，Spring Boot能够智能识别项目依赖、自动配置Bean，并结合类路径扫描确保所有相关的组件和服务都被正确地初始化和管理，我们就可以专注于业务逻辑的开发，而不必过多考虑基础设施层面的配置问题。

## 总结

Spring Boot 应用程序被打包成的jar包之所以可以直接通过 `java -jar` 命令运行，是因为Spring Boot在构建过程中做了一些特殊的设计和配置。具体原因：

1. **Fat/Uber JAR**: Spring Boot使用maven插件`spring-boot-maven-plugin`（或Gradle对应的插件）将项目及其所有依赖项打包成一个单一的、自包含的jar文件，通常称为“Fat JAR”或“Uber JAR”。这意味着不仅包含了自己的类文件，还包含了运行应用所需的所有第三方库。
2. **Manifest.MF**: 在打包过程中，此插件会修改MANIFEST.MF文件，这是jar包中的一个元数据文件。在MANIFEST.MF中，特别指定了`Main-Class`属性，该属性指向Spring Boot的一个内置的启动类（如`org.springframework.boot.loader.JarLauncher`），这个启动器类知道如何正确启动Spring Boot应用程序。
3. **嵌入式Servlet容器**：Spring Boot默认集成了诸如Tomcat、Jetty或Undertow等嵌入式Web容器，使得无需外部服务器环境也能运行Web应用。
4. **启动器类加载器**：当通过`java -jar`运行Spring Boot应用时，JVM会根据MANIFEST.MF中的`Main-Class`找到并运行指定的启动器类。这个启动器类加载器能够解压并加载内部的依赖库，并定位到实际的应用主类（在`spring-boot-starter-parent`或`@SpringBootApplication`注解标记的类），进而执行其`main`方法。
5. **类路径扫描和自动配置**：Spring Boot应用通过特定的类路径扫描机制和自动配置功能，能够在启动时识别出应用所依赖的服务和组件，并自动配置它们，大大简化了传统Java应用的配置和部署过程。

Spring Boot通过精心设计的打包流程和启动器类，使得生成的jar包可以直接作为一个独立的应用程序运行，极大地简化了部署和运维复杂度。

本文已收录于我的个人博客：[码农Academy的博客，专注分享Java技术干货，包括Java基础、Spring Boot、Spring Cloud、Mysql、Redis、Elasticsearch、中间件、架构设计、面试题、程序员攻略等](https://link.juejin.cn/?target=https%3A%2F%2Fwww.coderacademy.online%2F "https://www.coderacademy.online/")