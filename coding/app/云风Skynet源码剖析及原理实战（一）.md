

深度Linux

 _2023年11月21日 21:27_ _湖南_

Skynet 是一个基于C跟lua的开源服务端并发框架，这个框架是单进程多线程Actor模型。是一个轻量级的为在线游戏服务器打造的框架。

![](http://mmbiz.qpic.cn/mmbiz_png/dkX7hzLPUR0Ao40RncDiakbKx1Dy4uJicoqwn5GZ5r7zSMmpwHdJt32o95wdQmPZrBW038j8oRSSQllpnOUDlmUg/300?wx_fmt=png&wxfrom=19)

**深度Linux**

拥有15年项目开发经验及丰富教学经验，曾就职国内知名企业项目经理，部门负责人等职务。研究领域：Windows&Linux平台C/C++后端开发、Linux系统内核等技术。

181篇原创内容

公众号

## 一、skynet介绍

### 1.1简介

这个系统是单进程多线程模型。每个服务都是严格的被动的消息驱动的，以一个统一的 callback 函数的形式交给框架。框架从消息队列里调度出接收的服务模块，找到 callback 函数入口，调用它。服务本身在没有被调度时，是不占用任何 CPU 的。

skynet虽然支持集群，但是作者云风主张能用一个节点完成尽量用一个节点，因为多节点通信方面的开销太大，如果一共有 100 个 skynet 节点，在它们启动完毕后，会建立起 9900条通讯通道。

### 1.2特点

Skynet框架做两个必要的保证：

1. 一个服务的 callback 函数永远不会被并发。
    
2. 一个服务向另一个服务发送的消息的次序是严格保证的。
    

用多线程模型来实现它。底层有一个线程消息队列，消息由三部分构成：源地址、目的地址、以及数据块。框架启动固定的多条线程，每条工作线程不断从消息队列取到消息，调用服务的 callback 函数。

线程数应该略大于系统的 CPU 核数，以防止系统饥饿。（只要服务不直接给自己不断发新的消息，就不会有服务被饿死）

对于目前的点对点消息，要求发送者调用 malloc 分配出消息携带数据用到的内存；由接受方处理完后调用 free 清理（由框架来做）。这样数据传递就不需要有额外的拷贝了。

做为核心功能，Skynet 仅解决一个问题：

把一个符合规范的 C 模块，从动态库（so 文件）中启动起来，绑定一个永不重复（即使模块退出）的数字 id 做为其 handle 。模块被称为服务（Service），服务间可以自由发送消息。每个模块可以向 Skynet 框架注册一个 callback 函数，用来接收发给它的消息。每个服务都是被一个个消息包驱动，当没有包到来的时候，它们就会处于挂起状态，对 CPU 资源零消耗。如果需要自主逻辑，则可以利用 Skynet 系统提供的 timeout 消息，定期触发。

### 1.3Actor模型

Actor模型内部的状态由它自己维护即它内部数据只能由它自己修改(通过消息传递来进行状态修改)，所以使用Actors模型进行并发编程可以很好地避免这些问题，Actor由状态(state)、行为(Behavior)和邮箱(mailBox)三部分组成：

- 状态(state)：Actor中的状态指的是Actor对象的变量信息，状态由Actor自己管理，避免了并发环境下的锁和内存原子性等问题
    
- 行为(Behavior)：行为指定的是Actor中计算逻辑，通过Actor接收到消息来改变Actor的状态
    
- 邮箱(mailBox)：邮箱是Actor和Actor之间的通信桥梁，邮箱内部通过FIFO消息队列来存储发送方Actor消息，接受方Actor从邮箱队列中获取消息
    

Actor的基础就是消息传递，skynet中每个服务就是一个LUA虚拟机，就是一个Actor。

Actor模型好处

1. 事件模型驱动： Actor之间的通信是异步的，即使Actor在发送消息后也无需阻塞或者等待就能够处理其他事情。
    
2. 强隔离性： Actor中的方法不能由外部直接调用，所有的一切都通过消息传递进行的，从而避免了Actor之间的数据共享，想要观察到另一个Actor的状态变化只能通过消息传递进行询问。
    
3. 位置透明： 无论Actor地址是在本地还是在远程机上对于代码来说都是一样的。
    
4. 轻量性：Actor是非常轻量的计算单机，只需少量内存就能达到高并发。
    

## 二、skynet原理

### 2.1消息队列

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上图摘自Actor模型解析，每个Actor都有一个专用的MailBox来接收消息，这也是Actor实现异步的基础。当一个Actor实例向另外一个Actor发消息的时候，并非直接调用Actor的方法，而是把消息传递到对应的MailBox里，就好像邮递员，并不是把邮件直接送到收信人手里，而是放进每家的邮箱，这样邮递员就可以快速的进行下一项工作。所以在Actor系统里，Actor发送一条消息是非常快的。

```
struct message_queue {    struct spinlock lock;    uint32_t handle;    int cap;    int head;    int tail;    int release;    int in_global;    int overload;    int overload_threshold;    struct skynet_message *queue;    struct message_queue *next;};struct global_queue {    struct message_queue *head;    struct message_queue *tail;    struct spinlock lock;};static struct global_queue *Q = NULL;
```

struct spinlock是自旋锁，用来解决并发问题的。

skynet也实现了Actor模型，每个服务都有一个专用的MailBox用来接收消息，这个队列即struct message_queue结构，skynet有两种消息队列，每个服务有一个刚刚谈到的被称为次级消息队列，skynet还有一个全局消息队列即static struct global_queue *Q = NULL;，头尾指针分别指向一个次级队列，在skynet启动时初始化全局队列。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

struct message_queue结构体种有一个struct skynet_message *queue这个就是每个服务的次级消息队列，是来自其他服务需要本服务处理的消息，这是一个数组实现的队列，在往队列push消息的时候会检查队列是否已满，满了会扩容一倍，跟vector有点类似。

配置文件中有个配置是thread = 8，这个就是worker线程的个数，worker线程每次会从global_mq中pop一条次级消息队列（每条次级消息对应特定服务），并根据线程权重和次级消息队列的回调函数，对次级消息队列中的消息进行消费。处理完毕后，会从global_mq中再pop一条次级消息队列供下次调用，同时将本次使用的次级消息队列push回global_mq的尾部。

### 2.2模块

```
struct modules {    int count;    struct spinlock lock;    const char * path;// 用c编写的服务编译后so库路经，不配置默认是./cservice/?.so    struct skynet_module m[MAX_MODULE_TYPE]; //存放服务模块的数组};static struct modules * M = NULL;struct skynet_module { //单个模块结构    const char * name; //c服务名称，一般是指c服务的文件名    void * module; //访问so库的dl句柄，通过dlopen获得该句柄    skynet_dl_create create; //通过dlsym绑定so库中的xxx_create函数，调用create即调用xxx_create接口    skynet_dl_init init; //绑定xxx_init接口    skynet_dl_release release; //绑定xxx_release接口    skynet_dl_signal signal; //绑定xxx_signal接口};
```

M->path在初始化(skynet_module_init)时赋值，对应配置文件的cpath，不配置默认是./cservice/?.so。  
用c编写的服务编译成so库后放在cpath目录下，当创建一个ctx时(skynet_context_new)，通过名称在skynet_module里找对应的module(skynet_module_query)，如果M->m存有同名称的module，返回即可。  
如果第一次创建该名称的服务，先找到该名称对应的so库的路径，然后通过dlopen函数获取so库的访问句柄dl(try_open)，再通过dlsym函数获取so库中xxx_create, xxx_init,xxx_release, xxx_signal4个函数地址(open_sym)，将这些地址赋值给skynet_module->create, skynet_module->init, skynet_module->release, skynet_module->signal。

通常一个c服务需要提供这4个接口，定义这些接口时一定要以服务名称为前缀，通过下划线和函数名称连接起来：

- xxx_create：创建ctx过程中调用，通常是申请内存。返回该服务的实例inst，设置ctx->instance=inst，之后init，release，signal都需要用到该实例
    
- xxx_init：创建ctx期间调用，除了初始化，最主要的工作是向ctx注册callback函数（skynet_callback），之后ctx才能正确的处理收到的消息（调用callback函数）
    
- xxx_release：释放ctx时调用(skynet_context_release)
    
- xxx_signal：ctx收到信号时调用
    

最后将skynet_module保存到M->m里，之后创建同名称ctx就不用获取so库的访问句柄了。

### 2.3服务

kynet是以服务为主体进行运作的，服务称作为skynet_context(简称ctx)，是一个c结构，是skynet里最重要的结构，整个skynet的运作都是围绕ctx进行的。skynet_server提供的api主要分两大类：

- 1.对ctx的一系列操作，比如创建，删除ctx等
    
- 2.如何发送消息和处理自身的消息
    

ctx的结构如下，创建一个服务时，会构建一个skynet上下文skynet_context，然后把该ctx存放到handle_storage(skynet_handle.c)里。ctx有一个引用计数（ctx->ref）控制其生命周期，当引用计数为0，删除ctx，释放内存。

```
struct skynet_context { //一个skynet服务ctx的结构        void * instance; //每个ctx自己的数据块，不同类型ctx有不同数据结构，相同类型ctx数据结构相同，但具体的数据不一样，由指定module的create函数返回        struct skynet_module * mod; //保存module的指针，方便之后调用create,init,signal,release        void * cb_ud; //给callback函数调用第二个参数，可以是NULL        skynet_cb cb; //消息回调函数指针，通常在module的init里设置        struct message_queue *queue; //ctx自己的消息队列指针        FILE * logfile; //日志句柄        uint64_t cpu_cost;      // in microsec        uint64_t cpu_start;     // in microsec        char result[32]; //保存skynet_command操作后的结果        uint32_t handle; //标识唯一的ctx id        int session_id; //本方发出请求会设置一个对应的session，当收到对方消息返回时，通过session匹配是哪一个请求的返回        int ref; //引用计数，当为0，可以删除ctx        int message_count; //累计收到的消息数量        bool init; //标记是否完成初始化        bool endless; //标记消息是否堵住        bool profile; //标记是否需要开启性能监测        CHECKCALLING_DECL // 自旋锁};
```

为了统一对ctx操作的接口，采用指令的格式，定义了一系列指令(cmd_xxx)，cmd_launch创建一个新服务，cmd_exit服务自身退出，cmd_kill杀掉一个服务等，上层统一调用skynet_command接口即可执行这些操作。对ctx操作，通常会先调用skynet_context_grab将引用计数+1，操作完调用skynet_context_release将引用计数-1，以保证操作ctx过程中，不会被其他线程释放掉。下面介绍几个常见的操作：

```
struct command_func { //skynet指令结构        const char *name;        const char * (*func)(struct skynet_context * context, const char * param);};static struct command_func cmd_funcs[] = { //skynet可接收的一系列指令        { "TIMEOUT", cmd_timeout },        { "REG", cmd_reg },        { "QUERY", cmd_query },        { "NAME", cmd_name },        { "EXIT", cmd_exit },        { "KILL", cmd_kill },        { "LAUNCH", cmd_launch },        { "GETENV", cmd_getenv },        { "SETENV", cmd_setenv },        { "STARTTIME", cmd_starttime },        { "ABORT", cmd_abort },        { "MONITOR", cmd_monitor },        { "STAT", cmd_stat },        { "LOGON", cmd_logon },        { "LOGOFF", cmd_logoff },        { "SIGNAL", cmd_signal },        { NULL, NULL },};
```

cmd_launch，启动一个新服务，最终会通过skynet_context_new创建一个ctx，初始化ctx中各个数据：

```
struct skynet_context * skynet_context_new(const char * name, const char *param) { //启动一个新服务ctx        struct skynet_module * mod = skynet_module_query(name); //从skynet_module获取对应的模板        ...        void *inst = skynet_module_instance_create(mod); //ctx独有的数据块，最终会调用c服务里的xxx_create        ...        ctx->mod = mod;        ctx->instance = inst;        ctx->ref = 2; //初始化完成会调用skynet_context_release将引用计数-1，ref变成1而不会被释放掉        ...        // Should set to 0 first to avoid skynet_handle_retireall get an uninitialized handle        ctx->handle = 0;                ctx->handle = skynet_handle_register(ctx); //从skynet_handle获得唯一的标识id        struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle); //初始化次级消息队列        ...        CHECKCALLING_BEGIN(ctx)        int r = skynet_module_instance_init(mod, inst, ctx, param);//初始化ctx独有的数据块        CHECKCALLING_END(ctx)}
```

在skynet的main函数中有config.bootstrap = optstring("bootstrap","snlua bootstrap")，随后bootstrap(ctx, config->bootstrap)，在这里会启动一个snlua服务，将snlua.so模块加载进来，snlua 是lua的沙盒服务，所有的 lua服务 都是一个 snlua 的实例。

snlua 实例化的过程：

这里我们来看一下 snlua 模块的实例化方法，源码在 service-src/service_snlua.c 中的 snlua_create(void) 函数：

```
struct snlua * snlua_create(void) {    struct snlua * l = skynet_malloc(sizeof(*l));    memset(l,0,sizeof(*l));    l->mem_report = MEMORY_WARNING_REPORT;    l->mem_limit = 0;    //创建一个lua虚拟机（Lua State）    l->L = lua_newstate(lalloc, l);    return l;}
```

最后返回的是一个通过 lua_newstate 创建出来的 Lua vm（lua虚拟机），也就是一个沙盒环境，这是为了达到让每个 lua服务 都运行在独立的虚拟机中。

上面的实例化步骤，只是生成了 lua服务 的运行沙盒环境，至于沙盒内运行的具体内容，是在初始化的时候才填充进来的，这里我们再来简单剖析一下初始化函数 snlua_init 的源码：

```
int snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {    int sz = strlen(args);    //在内存中准备一个空间（动态内存分配）    char * tmp = skynet_malloc(sz);    //内存拷贝：将args内容拷贝到内存中的temp指针指向地址的内存空间    memcpy(tmp, args, sz);    //注册回调函数为launch_cb这个函数，有消息传入时会调用回调函数并处理    skynet_callback(ctx, l , launch_cb);    const char * self = skynet_command(ctx, "REG", NULL);    //当前lua实例自己的句柄id（转为无符号长整型）    uint32_t handle_id = strtoul(self+1, NULL, 16);    // it must be first message    // 给自己发送一条消息，内容为args字符串    skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);    return 0;}
```

这个初始化函数主要完成了两件事：

- 给当前服务实例注册绑定了一个回调函数 launch_cb；
    
- 给本服务发送一条消息，内容就是之前传入的参数 bootstrap 。
    

当此服务的消息队列被push进全局的消息队列后，本服务收到的第一条消息就是上述在初始化中给自己发送的那条消息，此时便会调用回调函数launch_cb并执行处理逻辑：

```
static int launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {    assert(type == 0 && session == 0);    struct snlua *l = ud;    //将服务原本绑定的句柄和回调函数清空    skynet_callback(context, NULL, NULL);    //设置各项资源路径参数，并加载loader.lua    int err = init_cb(l, context, msg, sz);    if (err) {        skynet_command(context, "EXIT", NULL);    }    return 0;}
```

这个方法里把服务自己在C语言层面的回调函数给注销了，使它不再接收消息，目的是：在lua层重新注册它，把消息通过lua接口来接收。

紧接着执行init_cb方法：

设置了一些虚拟机环境变量（主要是资源路径类的）同时把真正要加载的文件（此时是 bootstrap.lua）作为参数传给它，最终 bootstrap.lua 脚本会被加载并执行脚本中的逻辑， 控制权就开始转到lua层。

在bootstrap.lua执行了 skynet.start 这个接口，这也是所有 lua服务 的标准启动入口，服务启动完成后，就会调用这个接口，传入的参数就是一个function（方法），而且这个方法就是此 lua服务 的在lua层的回调接口，本服务的消息都在此回调方法中执行。

skynet.start 接口

关于每个lua服务的启动入口 skynet.start 接口的实现代码在 service/skynet.lua 中：

```
function skynet.start(start_func)    --重新注册一个callback函数，并且指定收到消息时由dispatch_message分发    c.callback(skynet.dispatch_message)    skynet.timeout(0, function()        skynet.init_service(start_func)    end)end
```

具体如何实现回调方法的注册过程，需要查看c.callback这个C语言方法的底层实现，源码在 lualib-src/lua-skynet.c：

```
static int lcallback(lua_State *L) {    struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));    int forward = lua_toboolean(L, 2);    luaL_checktype(L,1,LUA_TFUNCTION);    lua_settop(L,1);    lua_rawsetp(L, LUA_REGISTRYINDEX, _cb);    lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_MAINTHREAD);    lua_State *gL = lua_tothread(L,-1);    if (forward) {        skynet_callback(context, gL, forward_cb);    } else {        skynet_callback(context, gL, _cb);    }    return 0;}
```

与上面snlua初始化中的一致，使用 skynet_callback 来实现回调方法的注册。

## 三、搭建skynet

### 3.1在ubuntu上搭建skynet

(1)获取skynet源代码（安装git代码管理工具）

```
$ sudo apt-get update$ sudo apt-get install git  
```

注意：如果安装失败，请先安装一下只支持库

```
$ sudo apt-get install build-essential libssl-dev libcurl4-gnutls-dev libexpat1-dev gettext unzip
```

(2)到github上面下载skynet的源代码 skynet的代码保存在github上面，大家可以去上面查看，现在我们用git把代码拷贝一份下来：

```
 $ git clone https://github.com/cloudwu/skynet.git
```

### 3.2skynet代码目录结构

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

```
3rd         #第三方支持库，包括LUA虚拟机，jmalloc等lualib      #lua语言封装的常用库，包括http、md5lualib-src  #将c语言实现的插件捆绑成lua库，例如数据库驱动、bson、加密算法等service     #使用lua写的Skynet的服务模块service-src #使用C写的Skynet的服务模块skynet-src  #skynet核心代码目录test        #使用lua写的一些测试代码examples    #示例代码Makefile    #编译规则文件，用于编译platform.mk #编译与平台相关的设置
```

### 3.3编译与运行skynet服务器

(1)编译skynet

```
$ cd skynet  #今后我们所有的工作都在这个目录中进行$ make linux#如果报错： ./autogen.sh: 5: ./autogen.sh: autoconf: not found#安装autoconf$ sudo apt-get install autoconf#如果报错：lua.c:83:31: fatal error: readline/readline.h: No such file or directory#安装libreadline-dev$ sudo apt-get install libreadline-dev#编译成功出现以下提示make[1]: Leaving directory '/home/ubuntu/workspace/skynet'#并且在目录里出现一个可执行文件skynet
```

(2)运行第一个skynet节点

```
 #启动一个skynet服务节点 $ ./skynet examples/config 
```

### 3.4运行客户端

我们要运行的的客户端是example/client.lua 这个lua脚本文件，那么首先你要有一个lua虚拟机程序。

(1)编译lua虚拟机

```
#打开另一个终端，开始编译虚拟机$ cd ./3rd/lua/ $ make linux#编译成功则会在当前路径上面看到一个可执行文件lua
```

(2)运行客户端

```
#跑到skynet根目录$ cd ../../#运行client.lua这个脚本$ ./3rd/lua/lua examples/client.lua 
```

## 四、构建服务的基础API

```
local skynet = require "skynet" --conf配置信息已经写入到注册表中，通过该函数获取注册表的变量值skynet.getenv(varName) 。--设置注册表信息，varValue一般是number或string，但是不能设置已经存在的varnameskynet.setenv(varName, varValue) --打印函数skynet.error(...)--用 func 函数初始化服务，并将消息处理函数注册到 C 层，让该服务可以工作。skynet.start(func) --若服务尚未初始化完成，则注册一个函数等服务初始化阶段再执行；若服务已经初始化完成，则立刻运行该函数。skynet.init(func) --结束当前服务skynet.exit() --获取当前服务的句柄handlerskynet.self()--将handle转换成字符串skynet.address(handler)--退出skynet进程require "skynet.manager"   --除了需要引入skynet包以外还要再引入skynet.manager包。skynet.abort()--强制杀死其他服务skynet.kill(address) --可以用来强制关闭别的服务。但强烈不推荐这样做。因为对象会在任意一条消息处理完毕后，毫无征兆的退出。所以推荐的做法是，发送一条消息，让对方自己善后以及调用 skynet.exit 。注：skynet.kill(skynet.self()) 不完全等价于 skynet.exit() ，后者更安全。
```

### 4.1编写一个test服务

(1)编写一个最简单的服务test.lua

```
--引入或者说是创建一个skynet服务local skynet = require "skynet" --调用skynet.start接口，并定义传入回调函数skynet.start(function()skynet.error("Server First Test")end)
```

(2)修改exmaple/config文件中的start的值为test，表示启动test.lua，修改之前请备份

```
include "config.path"-- preload = "./examples/preload.lua"   -- run preload.lua before every lua service runthread = 2logger = nillogpath = "."harbor = 1address = "127.0.0.1:2526"master = "127.0.0.1:2013"start = "test"  -- main script  --将start的值修改为testbootstrap = "snlua bootstrap"   -- The service for bootstrapstandalone = "0.0.0.0:2013"-- snax_interface_g = "snax_g"cpath = root.."cservice/?.so"-- daemon = "./skynet.pid"
```

(3)通过skynet来运行test.lua

```
$ ./skynet examples/config
```

运行结果

```
$ ./skynet examples/config[:01000001] LAUNCH logger [:01000002] LAUNCH snlua bootstrap[:01000003] LAUNCH snlua launcher[:01000004] LAUNCH snlua cmaster[:01000004] master listen socket 0.0.0.0:2013[:01000005] LAUNCH snlua cslave[:01000005] slave connect to master 127.0.0.1:2013[:01000004] connect from 127.0.0.1:52132 4[:01000006] LAUNCH harbor 1 16777221[:01000004] Harbor 1 (fd=4) report 127.0.0.1:2526[:01000005] Waiting for 0 harbors[:01000005] Shakehand ready[:01000007] LAUNCH snlua datacenterd[:01000008] LAUNCH snlua service_mgr[:01000009] LAUNCH snlua test[:01000009] Server First Test[:01000002] KILL self
```

注意：千万不要在skynet根目录以外的地方执行skynet，例如：

```
$ cd examples$ ../skynet configtry open logger failed : ./cservice/logger.so: cannot open shared object file: No such file or directoryCan't launch logger service$ 
```

以上出现找不到logger.so的情况，其实不仅仅是这个模块找不到，所有的模块都找不到了，因为在config包含的路劲conf.path中，所有的模块路劲的引入全部依靠着相对路劲。一旦执行skynet程序的位置不一样了，相对路劲也会不一样。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

(4)添加自己的LUA脚本路劲

例如：添加my_workspace目录，则只需在luaservice值的基础上再添加一个`root.."my_workspace/？.lua;"`,注意：各个路劲通过一个`;` 隔开。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

把我们的刚才写的test.lua丢到my_workspace中

```
$ mv examples/test.lua my_workspace/
```

顺便将example下的conf以及conf.path也拷贝一份到my_workspace

```
$ cp examples/config my_workspace/$ cp examples/config.path my_workspace/
```

这次运行的时候就可以这样了：

```
$ ./skynet my_workspace/conf
```

### 4.2另外一个种启动服务的方式

另一种方式启动想要的服务，可以在main.lua运行后，在console直接输入需要启动的服务名称.

1.先启动main.lua服务，注意还原examples/conf默认配置，并且在example/conf.path添加自己的服务目录。

```
 $ ./skynet examples/conf  #conf中start配置为启动main.lua   [:01000001] LAUNCH logger    [:01000002] LAUNCH snlua bootstrap   [:01000003] LAUNCH snlua launcher   [:01000004] LAUNCH snlua cmaster   [:01000004] master listen socket 0.0.0.0:2013   [:01000005] LAUNCH snlua cslave   [:01000005] slave connect to master 127.0.0.1:2013   [:01000004] connect from 127.0.0.1:52698 4   [:01000006] LAUNCH harbor 1 16777221   [:01000004] Harbor 1 (fd=4) report 127.0.0.1:2526   [:01000005] Waiting for 0 harbors   [:01000005] Shakehand ready   [:01000007] LAUNCH snlua datacenterd   [:01000008] LAUNCH snlua service_mgr   [:01000009] LAUNCH snlua main   [:01000009] Server start   [:0100000a] LAUNCH snlua protoloader   [:0100000b] LAUNCH snlua console   [:0100000c] LAUNCH snlua debug_console 8000   [:0100000c] Start debug console at 127.0.0.1:8000   [:0100000d] LAUNCH snlua simpledb   [:0100000e] LAUNCH snlua watchdog   [:0100000f] LAUNCH snlua gate   [:0100000f] Listen on 0.0.0.0:8888   [:01000009] Watchdog listen on 8888   [:01000009] KILL self   [:01000002] KILL self
```

2.在启动的main服务中，直接输入test，回车

```
$ ./skynet examples/config   [:01000001] LAUNCH logger    [:01000002] LAUNCH snlua bootstrap   [:01000003] LAUNCH snlua launcher   [:01000004] LAUNCH snlua cmaster   [:01000004] master listen socket 0.0.0.0:2013   [:01000005] LAUNCH snlua cslave   [:01000005] slave connect to master 127.0.0.1:2013   [:01000004] connect from 127.0.0.1:52698 4   [:01000006] LAUNCH harbor 1 16777221   [:01000004] Harbor 1 (fd=4) report 127.0.0.1:2526   [:01000005] Waiting for 0 harbors   [:01000005] Shakehand ready   [:01000007] LAUNCH snlua datacenterd   [:01000008] LAUNCH snlua service_mgr   [:01000009] LAUNCH snlua main   [:01000009] Server start   [:0100000a] LAUNCH snlua protoloader   [:0100000b] LAUNCH snlua console   [:0100000c] LAUNCH snlua debug_console 8000   [:0100000c] Start debug console at 127.0.0.1:8000   [:0100000d] LAUNCH snlua simpledb   [:0100000e] LAUNCH snlua watchdog   [:0100000f] LAUNCH snlua gate   [:0100000f] Listen on 0.0.0.0:8888   [:01000009] Watchdog listen on 8888   [:01000009] KILL self   [:01000002] KILL self   test    #终端输入   [:01000010] LAUNCH snlua test    [:01000010] Server First Test   #服务已经启动
```

### 4.3环境变量

- 1、预先加载的环境变量是在conf中配置的，加载完成后，所有的service都能去获取这些变量。
    
- 2、也可以去设置环境变量，但是不能修改已经存在的环境变量。
    
- 3、环境变量设置完成后，当前节点上的所有服务都能访问的到。
    
- 4、环境变量设置完成后，及时服务退出了，环境变量依然存在，所以不要滥用环境变量。
    

例如在conf中添加：

```
myname = "Dmaker"myage = 20
```

示例代码：testenv.lua

```
local skynet = require "skynet" skynet.start(function()    --获取环境变量myname和myage的值，成功返回其值，如果该环境变量不存在返回nil    local name = skynet.getenv("myname")        local age = skynet.getenv("myage")      skynet.error("My name is", name, ",", age, "years old.")        --skynet.setenv("myname", "coder")  --不要尝试设置已经存在的变量值，会报错    --skynet.setenv("myage", 21)    skynet.setenv("mynewname", "coder") --设置一个新的变量    skynet.setenv("mynewage", 21)    name = skynet.getenv("mynewname")       age = skynet.getenv("mynewage")     skynet.error("My new name is", name, ",", age, "years old soon.")    skynet.exit()end)
```

### 4.4skynet.init的使用

skynet.init用来注册服务初始化之前，需要执行的函数。也就是在skynet.start之前运行。

示例代码：testinit.lua

```
local skynet = require "skynet" skynet.init(function()        skynet.error("service init")end)skynet.start(function()    skynet.error("service start")end)
```

运行结果：

```
[:01000009] service init    #先运行skynet.init[:01000009] service start   #运行skynet.start函数
```

## 五、skynet服务类型

skynet中的服务分为普通服务与全局唯一服务。第3节启动方式就是一个普通服务，而全局唯一服务顾名思义就是在skynet中只能生成一个服务实例。

### 5.1普通服务

每调用一次创建接口就会创建出一个对应的服务实例，可以同时创建成千上万个，用唯一的id来区分每个服务实例。使用的创建接口是：

```
--[[1. 用于启动一个新的 Lua 服务,luaServerName是脚本的名字（不用写 .lua 后缀）。2. 只有被启动的脚本的 start 函数返回后，这个 API 才会返回启动的服务的地址，这是一个阻塞 API 。3. 如果被启动的脚本在初始化环节抛出异常,skynet.newservice 也会执行失败。4. 如果被启动的脚本的 start 函数是一个永不结束的循环，那么 newservice 也会被永远阻塞住。--]]skynet.newservice(luaServerName, ...)
```

(1)启动一个test服务testnewservice.lua

```
local skynet = require "skynet" --调用skynet.start接口，并定义传入回调函数skynet.start(function()    skynet.error("My new service")    skynet.newservice("test")    skynet.error("new test service")end)
```

启动main.lua，并且输入testnewservice

```
$ ./skynet examples/config #默认启动main.lua服务testnewservice[:01000010] LAUNCH snlua testnewservice  #通过main服务，启动testnewservice服务[:01000010] My new service           [:01000012] LAUNCH snlua test           #再启动test服务[:01000012] Server First Test[:01000010] new test service
```

(2)启动两个test服务testnewservice2.lua

```
local skynet = require "skynet" --调用skynet.start接口，并定义传入回调函数skynet.start(function() skynet.error("My new service") skynet.newservice("test") skynet.error("new test service 0") skynet.newservice("test") skynet.error("new test service 1")end)
```

启动main.lua，并且输入testnewservice

```
$ ./skynet examples/config  #默认启动main.lua服务testnewservice          #终端输入[:01000010] LAUNCH snlua testnewservice  #通过main服务，启动testnewservice服务[:01000010] My new service[:01000012] LAUNCH snlua test     #启动一个test服务[:01000012] Server First Test[:01000010] new test service 0[:01000019] LAUNCH snlua test     #再启动一个test服务[:01000019] Server First Test[:01000010] new test service 1
```

### 5.2全局唯一服务

全局唯一的服务等同于单例，即不管调用多少次创建接口，最后都只会创建一个此类型的服务实例，且全局唯一。

- 创建接口：
    

```
skynet.uniqueservice(servicename, ...) skynet.uniqueservice(true, servicename, ...) 
```

- 当带参数 `true` 时，则表示此服务在所有节点之间是唯一的。第一次创建唯一服，返回服务地址，第二次创建的时候不会正常创建服务，只是返回第一次创建的服务地址。
    
- 查询接口： 假如不清楚当前创建了此全局服务没有，可以通过以下接口来查询：
    

```
skynet.queryservice(servicename, ...) skynet.queryservice(true, servicename, ...) 
```

如果还没有创建过目标服务则一直等下去，直到目标服务被(其他服务触发而)创建。当参数 带`true` 时，则表示查询在所有节点中唯一的服务是否存在。

(1)测试skynet.uniqueservice接口

示例：testunique.lua

```
local skynet = require "skynet" local args = { ... }if(#args == 0) then    table.insert(args, "uniqueservice")endskynet.start(function()    local us    skynet.error("test unique service")    if ( #args == 2 and args[1] == "true" )  then        us = skynet.uniqueservice(true, args[2])    else        us =skynet.uniqueservice(args[1])    end    skynet.error("uniqueservice handler:", skynet.address(us))end)
```

示例：uniqueservice.lua

```
local skynet = require "skynet" skynet.start(function()    skynet.error("Server First Test")    --skynet.exit() 不要尝试服务初始化阶段退出服务，唯一服会创建失败end)
```

运行结果：

```
$ ./skynet examples/configtestunique   #终端输入[:01000010] LAUNCH snlua testunique[:01000010] test unique service[:01000012] LAUNCH snlua uniqueservice  #第一次创建全局唯一服uniqueservice成功[:01000012] unique service start[:01000010] uniqueservice handler: :01000012testunique  #终端输入[:01000019] LAUNCH snlua testunique [:01000019] test unique service[:01000019] uniqueservice handler: :01000012  #第二次创建并没有创建全局唯一服
```

(2)测试skynet.queryservice接口

示例：testqueryservice.lua

```
local skynet = require "skynet" local args = { ... }if(#args == 0) then    table.insert(args, "uniqueservice")endskynet.start(function()    local us    skynet.error("start query service")    --如果test服务未被创建，该接口将会阻塞，后面的代码将不会执行    if ( #args == 2 and args[1] == "true" )  then        us = skynet.queryservice(true, args[2])    else        us = skynet.queryservice(args[1])      end    skynet.error("end query service handler:", skynet.address(us))end)
```

运行结果：如果不启动test全局唯一服务，直接执行查询函数

```
$ ./skynet examples/configtestqueryservice   #终端输入[:01000010] LAUNCH snlua testqueryservice[:01000010] start query service            #阻塞住，不再执行后面的代码
```

启动test全局唯一服务，再执行查询函数

```
$ ./skynet examples/configtestunique  #终端输入[:01000010] LAUNCH snlua testunique[:01000010] test unique service[:01000012] LAUNCH snlua uniqueservice #第一次创建全局唯一服成功[:01000012] Server First Test[:01000010] uniqueservice handler: :01000012testqueryservice[:01000019] LAUNCH snlua testqueryservice   #再启动查询[:01000019] start query service[:01000019] end query service handler: :01000012 #skynet.queryservice将会返回
```

注意：当调用uniqueservice只传一个服务名时，表示创建当前skynet节点的全局唯一服务。当第一个参数传递true，第二个参数传递服务名时，则表示所有节点的全局唯一服务。

调用queryservice时，也可以选择是否传递第一个参数true， 表示查询的是当前skynet节点的全局唯一服，还是所有节点的全局唯一服。这两种全局唯一服作用范围是不同，所有可以同时存在同名的但作用范围不同的全局唯一服。

### 5.3多节点中的全局服务

首先，我们先启动两个节点出来。copy两个份`examp/config`为`config1`与`config2`, config1中修改如下：

config1：

```
include "config.path"-- preload = "./examples/preload.lua"   -- run preload.lua before every lua service runthread = 2 logger = nillogpath = "."harbor = 1      --表示每个节点编号address = "127.0.0.1:2526"master = "127.0.0.1:2013"start = "console"   -- main script 只启动一个console.lua服务bootstrap = "snlua bootstrap"   -- The service for bootstrapstandalone = "0.0.0.0:2013" --主节点才会用到这个，绑定地址-- snax_interface_g = "snax_g"cpath = root.."cservice/?.so"-- daemon = "./skynet.pid"
```

config2：

```
include "config.path"-- preload = "./examples/preload.lua"   -- run preload.lua before every lua service runthread = 2 logger = nillogpath = "."harbor = 2      --编号需要改address = "127.0.0.1:2527"   --改一个跟config1不同的端口master = "127.0.0.1:2013"   --主节点地址不变start = "console"   -- main scriptbootstrap = "snlua bootstrap"   -- The service for bootstrap--standalone = "0.0.0.0:2013" --作为从节点，就注释掉这里-- snax_interface_g = "snax_g"cpath = root.."cservice/?.so"-- daemon = "./skynet.pid"
```

启动两个终端分别启动如下：

节点1启动：

```
./skynet examples/config1 [:01000001] LAUNCH logger [:01000002] LAUNCH snlua bootstrap[:01000003] LAUNCH snlua launcher[:01000004] LAUNCH snlua cmaster  #启动主节点cmaster服务[:01000004] master listen socket 0.0.0.0:2013  #监听端口2013[:01000005] LAUNCH snlua cslave #主节点也要启动一个cslave，去连接cmaster节点[:01000005] slave connect to master 127.0.0.1:2013 #cslave中一旦连接完cmaster就会启动一个harbor服务[:01000004] connect from 127.0.0.1:47660 4 [:01000006] LAUNCH harbor 1 16777221 #cslave启动一个Harbor服务 用于节点间通信[:01000004] Harbor 1 (fd=4) report 127.0.0.1:2526 #报告cmaster cslave服务的地址[:01000005] Waiting for 0 harbors #cmaster告诉cslave还有多少个其他cslave需要连接[:01000005] Shakehand ready         #cslave与cmaster握手成功[:01000007] LAUNCH snlua datacenterd[:01000008] LAUNCH snlua service_mgr[:01000009] LAUNCH snlua console[:01000002] KILL self[:01000004] connect from 127.0.0.1:47670 6 #cmaster收到其他cslave连接请求[:01000004] Harbor 2 (fd=6) report 127.0.0.1:2527  #其他cslave报告地址[:01000005] Connect to harbor 2 (fd=7), 127.0.0.1:2527 #让当前cslave去连接其他cslave
```

节点2启动：

```
./skynet examples/config2 [:02000001] LAUNCH logger [:02000002] LAUNCH snlua bootstrap[:02000003] LAUNCH snlua launcher[:02000004] LAUNCH snlua cslave[:02000004] slave connect to master 127.0.0.1:2013 #cslave去连接主节点的cmaster服务[:02000005] LAUNCH harbor 2 33554436 #cslave也启动一个harbor服务[:02000004] Waiting for 1 harbors   #等待主节点的cslave来连接[:02000004] New connection (fd = 3, 127.0.0.1:37470) #cslave与主节点cslave连接成功[:02000004] Harbor 1 connected (fd = 3)[:02000004] Shakehand ready   #cslave与cmaster握手成功[:02000006] LAUNCH snlua service_mgr[:02000007] LAUNCH snlua console[:02000002] KILL self
```

（1）测试全节点全局唯一服

在第一个节点中启动testunique.lua服务，然后第二个节点中启动testqueryservice.lua服务查询

节点1：

```
testunique true uniqueservice #所有节点全局唯一服方式启动[:0100000b] LAUNCH snlua testunique true uniqueservice[:0100000b] test unique service[:0100000c] LAUNCH snlua uniqueservice[:0100000c] Server First Test[:0100000b] uniqueservice handler: :0100000c
```

节点2：

```
testqueryservice true uniqueservice[:02000012] LAUNCH snlua testqueryservice true uniqueservice[:02000012] start query service[:02000012] end query service handler: :0100000b#节点1中已经启动了，所以节点2中
```

(3)本地全局唯一服与全节点全局唯一服区别

节点2还可以创建一个同脚本的本地全局唯一服：

```
testunique uniqueservice[:0100000c] LAUNCH snlua testunique [:0100000c] test unique service[:0100000d] LAUNCH snlua uniqueservice[:0100000d] Server First Test[:0100000c] uniqueservice handler: :0100000d #创建了一个本地全局唯一服
```

但是无法创建一个新的全节点全局唯一服：

```
testunique true uniqueservice[:0100000e] LAUNCH snlua testunique true uniqueservice[:0100000e] test unique service[:0100000e] uniqueservice handler: :0100000b #还是节点1的全局唯一服句柄
```

## 六、服务别名

每个服务启动之后，都有一个整形数来表示id，也可以使用字符串id来表示，例如：:01000010，其实就是把id：0x01000010转换成字符串。

但是这个数字的表示方式会根据服务的启动先后顺序而变化，不是一个固定的值。如果想要方便的获取某个服务，那么可以通过给服务设置别名来。

### 6.1本地别名与全局别名

在skynet中，服务别名可以分为两种：

- 一种是本地别名，本地别名只能在当前skynet节点使用，本地别名必须使用`.` 开头，例如：`.testalias`
    
- 一种是全局别名，全局别名可以在所有skynet中使用，全局别名不能以`.` 开头， 例如：`testalias`
    

### 6.2别名注册与查询接口

```
------------------------------[[取别名]]--------------------------local skynet = require "skynet"require "skynet.manager"--给当前服务定一个别名，可以是全局别名，也可以是本地别名skynet.register(aliasname)--给指定servicehandler的服务定一个别名，可以是全局别名，也可以是本地别名skynet.name(aliasname, servicehandler)-------------------------------------------------------------------
```

  

```
----------------------[[查询别名]]--------------------------------查询本地别名为aliasname的服务，返回servicehandler，不存在就返回nilskynet.localname(aliasname)--[[查询别名为aliasname的服务,可以是全局别名也可以是本地别名，1、当查询本地别名时，返回servicehandler，不存在就返回nil2、当查询全局别名时，返回servicehandler，不存在就阻塞等待到该服务初始化完成]]--local skynet = require "skynet.harbor"harbor.queryname(aliasname)-------------------------------------------------------------------
```

注意：本地别名与全局别名可以同时存在。

### 6.3给服务注册别名

(1)给普通服取别名

示例代码：testalias.lua

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()    local handle = skynet.newservice("test")    skynet.name(".testalias", handle)   --给服务起一个本地别名    skynet.name("testalias", handle)    --给服务起一个全局别名            handle = skynet.localname(".testalias")        skynet.error("localname .testalias handle", skynet.address(handle))        handle = skynet.localname("testalias")      --只能查本地，不能查全局别名          skynet.error("localname testalias handle", skynet.address(handle))        handle = harbor.queryname(".testalias")                 skynet.error("queryname .testalias handle", skynet.address(handle))        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))  end)
```

上面服务通过skynet.newservice来启动一个test.lua服务，test.lua代码如下：

```
local skynet = require "skynet" skynet.start(function()    skynet.error("My new service")end)
```

运行结果：

```
$ ./skynet examples/configtestalias #运行main.lua后在终端输入[:0100000a] LAUNCH snlua testalias[:0100000b] LAUNCH snlua test[:0100000b] My new service[:0100000a] localname .testalias handle :0100000b   #skynet.localname查到本地名[:0100000a] localname testalias handle nil          #skynet.localname查不到全局名[:0100000a] queryname .testalias handle :0100000b   #harbor.queryname查到本地名[:0100000a] queryname testalias handle :0100000b    #harbor.queryname查到全局名
```

(2)全局别名查询阻塞

如果全局别名不存在，那么这个时候调用函数`harbor.queryname`，将会阻塞，直到全局别名的服务创建成功。

示例代码：testalias.lua

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()            handle = skynet.localname(".testalias")    --查询本地别名不阻塞    skynet.error("localname .testalias handle", skynet.address(handle))        handle = skynet.localname("testalias")      --无法查询全局别名    skynet.error("localname testalias handle", skynet.address(handle))        handle = harbor.queryname(".testalias")     --查询本地别名不阻塞     skynet.error("queryname .testalias handle", skynet.address(handle))        handle = harbor.queryname("testalias")      --查询全局别名阻塞           skynet.error("queryname testalias handle", skynet.address(handle))  end)
```

运行结果：

```
$ ./skynet examples/config                                             testalias[:0100000a] LAUNCH snlua testalias[:0100000a] localname .testalias handle nil #skynet.localname查到本地名[:0100000a] localname testalias handle nil  #skynet.localname查不到全局名[:0100000a] queryname .testalias handle nil #harbor.queryname查到本地名                                        #harbor.queryname查不到全局名, 函数阻塞
```

(3)多节点中的全局别名

启动两个skynet节点，在节点1取别名，节点2查询别名：

- 节点1，testaliasname.lua
    

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()    local handle = skynet.newservice("test")    skynet.name(".testalias", handle)   --给服务起一个本地别名    skynet.name("testalias", handle)    --给服务起一个全局别名end)
```

- 节点2, testaliasquery.lua
    

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()        handle = skynet.localname(".testalias")                  skynet.error("localname .testalias handle", skynet.address(handle))        handle = skynet.localname("testalias")                   skynet.error("localname testalias handle", skynet.address(handle))        handle = harbor.queryname(".testalias")                 skynet.error("queryname .testalias handle", skynet.address(handle))        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))end)
```

- 先启动节点1运行testaliasname.lua，再启动节点2运行
    

```
testaliasquery[:0200000a] LAUNCH snlua testaliasquery[:0200000a] localname .testalias handle nil[:0200000a] localname testalias handle nil[:0200000a] queryname .testalias handle nil[:0200000a] queryname testalias handle :0100000b --查询到节点1创建的服务
```

(4)杀死带别名的服务

给一个服务取了别名后，杀死它，本地别名将会注销掉，但是全局别名依然存在，通过全局别名查询到的handle已经没有意义。如果通过handle进行一些操作将得到不可预知的问题。

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()    local handle = skynet.newservice("test")    skynet.name(".testalias", handle)   --给服务起一个本地别名    skynet.name("testalias", handle)    --给服务起一个全局别名            handle = skynet.localname(".testalias")                  skynet.error("localname .testalias handle", skynet.address(handle))        handle = skynet.localname("testalias")                   skynet.error("localname testalias handle", skynet.address(handle))        handle = harbor.queryname(".testalias")                 skynet.error("queryname .testalias handle", skynet.address(handle))        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))    skynet.kill(handle) --杀死带别名服务    handle = skynet.localname(".testalias")                  skynet.error("localname .testalias handle", skynet.address(handle))        handle = skynet.localname("testalias")                   skynet.error("localname testalias handle", skynet.address(handle))        handle = harbor.queryname(".testalias")                 skynet.error("queryname .testalias handle", skynet.address(handle))        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))end)
```

运行结果：

```
testalias[:0100000a] LAUNCH snlua testalias[:0100000b] LAUNCH snlua test[:0100000b] My new service[:0100000a] localname .testalias handle :0100000b[:0100000a] localname testalias handle nil[:0100000a] queryname .testalias handle :0100000b[:0100000a] queryname testalias handle :0100000b[:0100000a] KILL :100000b[:0100000a] localname .testalias handle nil[:0100000a] localname testalias handle nil[:0100000a] queryname .testalias handle nil[:0100000a] queryname testalias handle :0100000b #全局别名还存在，但是已经杀死该服务了。
```

skynet的全局别名服务是在cslave里面实现的，现在不允许二次修改全局别名绑定关系，所以全局别名一般用来给一个永远不会退出的服务来启用。

但是有些情况下，我们确实需要二次修改全局别名绑定关系，那么这个时候，我们可以尝试去修改一下`cslave.lua`文件，修改内容如下：

```
function harbor.REGISTER(fd, name, handle)    --assert(globalname[name] == nil)  --将这一行注释掉    globalname[name] = handle    response_name(name)    socket.write(fd, pack_package("R", name, handle))    skynet.redirect(harbor_service, handle, "harbor", 0, "N " .. name)end
```

运行一个二次修改全局别名绑定关系的服务，例如：

```
local skynet = require "skynet" require "skynet.manager"local harbor = require "skynet.harbor"skynet.start(function()    local handle = skynet.newservice("test")    skynet.name("testalias", handle)    --给服务起一个全局别名        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))    skynet.kill(handle) --杀死带全局别名服务    handle = skynet.newservice("test")    skynet.name("testalias", handle)    --全局别名给其他服务使用        handle = harbor.queryname("testalias")                   skynet.error("queryname testalias handle", skynet.address(handle))end)
```

运行结果：

```
testalias[:0100000a] LAUNCH snlua testalias[:0100000b] LAUNCH snlua test[:0100000b] My new service[:0100000a] queryname testalias handle :0100000b[:0100000a] KILL :100000b[:0100000c] LAUNCH snlua test[:0100000c] My new service[:0100000a] queryname testalias handle :0100000c   #服务别名二次修改成功
```

### 6.4全局别名与全局唯一服名区别

全局唯一服名与这里的全局别名是两个概念的名词。

全局唯一服名称: 是用来标识服务是唯一的，服务名称一般就是脚本名称，无法更改。

全局别名: 是用来给服务起别名的，既可以给普通服起别名，也可以给全局唯一服起别名。

他们两种名字是在不同的体系中的，有各种的起名字的方式，以及查询的方式。

所以不要尝试用skynet.queryservice查询一个全局别名，也不要尝试使用harbor.queryname去查询一个全局唯一服。

例如：

```
local handle = skynet.uniqueservice("test") --启动一个全局唯一服，名字为testhandle = harbor.queryname("test")           --查不到的，会一直阻塞
```

或者：

```
local handle = skynet.uniqueservice("test") --启动一个全局唯一服，名字为testskynet.name("testalias", handle)            --再起一个全局别名handle = skynet.queryservice("testalias")   --查不到的，也会一直阻塞
```

## 七、服务调度

```
local skynet = require "skynet"--让当前的任务等待 time * 0.01s 。skynet.sleep(time)  --启动一个新的任务去执行函数 func , 其实就是开了一个协程，函数调用完成将返回线程句柄--虽然你也可以使用原生的coroutine.create来创建协程，但是会打乱skynet的工作流程skynet.fork(func, ...) --让出当前的任务执行流程，使本服务内其它任务有机会执行，随后会继续运行。skynet.yield()--让出当前的任务执行流程，直到用 wakeup 唤醒它。skynet.wait()--唤醒用 wait 或 sleep 处于等待状态的任务。skynet.wakeup(co)       --设定一个定时触发函数 func ，在 time * 0.01s 后触发。skynet.timeout(time, func)  --返回当前进程的启动 UTC 时间（秒）。skynet.starttime()  --返回当前进程启动后经过的时间 (0.01 秒) 。skynet.now()--通过 starttime 和 now 计算出当前 UTC 时间（秒）。skynet.time()           
```

### 7.1使用sleep休眠

示例代码：testsleep.lua

```
local skynet = require "skynet"skynet.start(function ()    skynet.error("begin sleep")    skynet.sleep(500)       skynet.error("begin end")end)
```

运行结果(还是先运行main.lua)：

```
$ ./skynet examples/configtestsleep                       #输入testsleep[:01000010] LAUNCH snlua testsleep[:01000010] begin sleep#执行权限已经被第一个服务testsleep给使用了，这里输入个新的服务test，并不会马上就启动服务test        [:01000010] begin end       [:01000012] LAUNCH snlua test  #等第一个服务完成任务了，才启动新的服务[:01000012] My new service
```

在console服务中输入testsleep之后，马上再输入test，会发现，test服务不会马上启动，因为这个时候 console正在忙于第一个服务testsleep初始化，需要等待5秒钟之后，输入的test 才会被console处理。

注意：以上做法是不正确的，在skynet.start函数中的服务初始化代码不允许有阻塞函数的存在，服务的初始化要求尽量快的执行完成，所有的业务逻辑代码不会写在skynet.start 里面。

### 7.2在服务中开启新的线程

在skynet的服务中我们可以开一个新的线程来处理业务（注意这个的线程并不是传统意义上的线程，更像是一个虚拟线程，其实是通过协程来模拟的）。

示例代码: testfork.lua

```
local skynet = require "skynet"function task(timeout)    skynet.error("fork co:", coroutine.running())    skynet.error("begin sleep")    skynet.sleep(timeout)        skynet.error("begin end")endskynet.start(function ()    skynet.error("start co:", coroutine.running())    skynet.fork(task, 500)  --开启一个新的线程来执行task任务    --skynet.fork(task, 500)  --再开启一个新的线程来执行task任务end)
```

运行结果：

```
$ ./skynet examples/configtestfork                        #输入testfork[:0100000a] LAUNCH snlua testfork   [:0100000a] start thread: 0x7f684d967568 false #不同的两个协程[:0100000a] fork  thread: 0x7f684d969f68 false[:0100000a] begin sleep
```

可以看到在testfork启动后，consloe服务仍然可以接受终端输入的test，并且启动。

以后如果遇到需要长时间运行，并且出现阻塞情况，都要使用skynet.fork在创建一个新的线程(协程)。

查看源码skynet.lua了解底层实现，其实就是使用coroutine.create实现，每次使用skynet.fork其实都是从协程池中获取未被使用的协程，并把该协程加入到fork队列中，等待一个消息调度，然后会依次把fork队列中协程拿出来执行一遍，执行结束后，会把协程重新丢入协程池中，这样可以避免重复开启关闭协程的额外开销。

### 7.3长时间占用执行权限的任务

示例代码：busytask.lua

```
local skynet = require "skynet"function task(name)    local i = 0    skynet.error(name, "begin task")    while ( i < 200000000)    do        i = i+1    end    skynet.error(name, "end task", i)endskynet.start(function ()    skynet.fork(task, "task1")    skynet.fork(task, "task2")end)
```

运行结果：

```
$ ./skynet examples/configbusytask[:01000010] LAUNCH snlua busytask[:01000010] task1 begin task       --先运行task1[:01000010] task1 end task 200000000[:01000010] task2 begin task        --再运行task2[:01000010] task2 end task 200000000
```

上面的运行结果充分说明了，`skynet.fork`创建的线程其实通过lua协程来实现的，即一个协程占用执行权后，其他的协程需要等待。

### 7.4使用skynet.yield让出执行权

示例代码：testyield.lua

```
local skynet = require "skynet"function task(name)    local i = 0    skynet.error(name, "begin task")    while ( i < 200000000)    do        i = i+1        if i % 50000000 == 0 then            skynet.yield()            skynet.error(name, "task yield")        end    end    skynet.error(name, "end task", i)endskynet.start(function ()    skynet.fork(task, "task1")    skynet.fork(task, "task2")end)
```

运行结果：

```
$ ./skynet examples/configtestyield[:01000010] LAUNCH snlua testyield[:01000010] task1 begin task[:01000010] task2 begin task[:01000010] task1 task yield[:01000010] task2 task yield[:01000010] task1 task yield[:01000010] task2 task yield[:01000010] task1 task yield[:01000010] task2 task yield[:01000010] task1 task yield[:01000010] task1 end task 200000000[:01000010] task2 task yield[:01000010] task2 end task 200000000
```

通过使用`skynet.yield()`然后同一个服务中的不同线程都可以得到执行权限。

### 7.5线程间的简单同步

同一个服务之间的线程可以通过，`skynet.wait`以及`skynet.wakeup`来同步线程

示例代码：testwakeup.lua

```
local skynet = require "skynet"local cos = {}function task1()    skynet.error("task1 begin task")    skynet.error("task1 wait")    skynet.wait()               --task1去等待唤醒    --或者skynet.wait(coroutine.running())    skynet.error("task1 end task")endfunction task2()    skynet.error("task2 begin task")    skynet.error("task2 wakeup task1")    skynet.wakeup(cos[1])           --task2去唤醒task1，task1并不是马上唤醒，而是等task2运行完    skynet.error("task2 end task")endskynet.start(function ()    cos[1] = skynet.fork(task1)  --保存线程句柄    cos[2] = skynet.fork(task2)end)
```

运行结果：

```
$ ./skynet examples/configtestwakeup[:01000010] LAUNCH snlua testwakeup[:01000010] task1 begin task[:01000010] task2 begin task[:01000010] task1 wait[:01000010] task2 wakeup task1[:01000010] task2 end task[:01000010] task1 end task
```

需要注意的是：skynet.wakeup除了能唤醒wait线程，也可以唤醒sleep的线程。

### 7.6定时器的使用

skynet中的定时器，其实是通过给定时器线程注册了一个超时时间，并且占用了一个空闲协程，空闲协程也是从协程池中获取，超时后会使用空闲协程来处理超时回调函数。

(1)启动一个定时器

示例代码：testtimeout.lua

```
local skynet = require "skynet"function task()    skynet.error("task", coroutine.running())endskynet.start(function ()     skynet.error("start", coroutine.running())    skynet.timeout(500, task) --5秒钟之后运行task函数,只是注册一下回调函数，并不会阻塞end)
```

运行结果：

```
$ ./skynet examples/configtesttimeout[:01000010] LAUNCH snlua testtimeout[:01000010] task
```

(2)skynet.start源码分析

其实skynet.start服务启动函数实现中，就已经启动了一个timeout为0s的定时器，来执行通过skynet.start函数传参得到的初始化函数。其目的是为了让skynet工作线程调度一次新服务。这一次服务调度最重要的意义在于把fork队列中的协程全部执行一遍。

(3)循环启动定时器

```
local skynet = require "skynet"function task()    skynet.error("task", coroutine.running())    skynet.timeout(500, task)endskynet.start(function ()  --skynet.start启动一个timeout来执行function，创建了一个协程     skynet.error("start", coroutine.running())    skynet.timeout(500, task) --由于function函数还没用完协程，所有这个timeout又创建了一个协程end)
```

执行结果，交替使用协程池中的协程：

```
testtimeout[:0100000a] LAUNCH snlua testtimeout[:0100000a] start thread: 0x7f525b16a048 false  #start函数也执行完，这个协程就空闲下来了[:0100000a] task thread: 0x7f525b16a128 false   #当前服务的协程池中只有两个协程，所以是交替使用[:0100000a] task thread: 0x7f525b16a048 false[:0100000a] task thread: 0x7f525b16a128 false[:0100000a] task thread: 0x7f525b16a048 false
```

### 7.7获取时间

示例代码：testtime.lua

```
local skynet = require "skynet"function task()    skynet.error("task")    skynet.error("start time", skynet.starttime()) --获取skynet节点开始运行的时间    skynet.sleep(200)    skynet.error("time", skynet.time())     --获取当前时间    skynet.error("now", skynet.now())       --获取skynet节点已经运行的时间endskynet.start(function ()    skynet.fork(task)end)
```

运行结果：

```
$ ./skynet examples/configtesttime[:01000010] LAUNCH snlua testtime[:01000010] task[:01000010] start time 1517161846[:01000010] time 1517161850.73[:01000010] now 473
```

### 7.8错误处理

lua中的错误处理都是通过assert以及error来抛异常，并且中断当前流程，skynet也不例外，但是你真的懂assert以及error吗？

注意这里的error不是skynet.error，skynet.error单纯写日志的，并不会中断流程。

skynet中使用assert或error后，服务会不会中断，skynet节点会不会中断？

来看一个例子：testassert.lua

```
local skynet = require "skynet"function task1()    skynet.error("task1", coroutine.running())    skynet.sleep(100)    assert(nil)    --error("error occurred")    skynet.error("task2", coroutine.running(), "end")endfunction task2()    skynet.error("task2", coroutine.running())    skynet.sleep(500)    skynet.error("task2", coroutine.running(), "end")endskynet.start(function ()      skynet.error("start", coroutine.running())    skynet.fork(task1)     skynet.fork(task2) end)
```

运行结果：

```
testassert[:0100000a] LAUNCH snlua testassert[:0100000a] start thread: 0x7fd9b12938e8 false[:0100000a] task1 thread: 0x7fd9b12939c8 false  [:0100000a] task2 thread: 0x7fd9b1293aa8 false[:0100000a] lua call [0 to :100000a : 2 msgsz = 0] error : ./lualib/skynet.lua:534: ./lualib/skynet.lua:156: ./my_workspace/testassert.lua:6: assertion failed!stack traceback:    [C]: in function 'assert'    ./my_workspace/testassert.lua:6: in function 'task1'    ./lualib/skynet.lua:468: in upvalue 'f'    ./lualib/skynet.lua:106: in function <./lualib/skynet.lua:105>stack traceback:    [C]: in function 'assert'    ./lualib/skynet.lua:534: in function 'skynet.dispatch_message'[:0100000a] task2 thread: 0x7fd9b1293aa8 end
```

上面的结果已经很明显了，开了两个协程分别执行task1、task2，task1断言后终止掉当前协程，不会再往下执行，但是task2还是能正常执行。skynet节点也没有挂掉，还是能正常运行。

那么我们在处理skynet的错误的时候可以大胆的使用assert与error，并不需要关注错误。当然，一个好的服务端，肯定不能一直出现中断掉的协程。

如果不想把协程中断掉，可以使用pcall来捕捉异常，例如：

```
local skynet = require "skynet"function task1()    skynet.error("task1", coroutine.running())    skynet.sleep(100)    assert(nil)    skynet.error("task2", coroutine.running(), "end")endfunction task2()    skynet.error("task2", coroutine.running())    skynet.sleep(500)    skynet.error("task2", coroutine.running(), "end")endskynet.start(function ()      skynet.error("start", coroutine.running())    skynet.fork(pcall, task1)     skynet.fork(pcall, task2) end)
```

运行结果：

```
testassert
```

C/C++开发92

Linux9

skynet1

C/C++开发 · 目录

上一篇Linux C/C++高级全栈开发知识树详解下一篇揭秘计算机408学习法，助你轻松成为计算机大神！

阅读 796

​