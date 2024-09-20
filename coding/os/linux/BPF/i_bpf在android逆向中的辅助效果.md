
lxcoder 看雪学苑

 _2024年03月16日 17:59_ _上海_

##   

一  

  

****背景****

  

样本是某比较流行的app，之前听说它有frida检测，正好有空就搂出来看看。  

  

  

二  

  

****初试（完结）****

  

frida检测常规的手段一般都是：

  

1.frida_server名字检测  
2.端口号检测  
3.D-Bus检测  
4.maps检测  
5.线程检测  
6.内存特征检测  

  

而1、2、3都有个特征，那就是在frida_server运行的时候才存在，不论是否附加进程，因为这个app是在附加时退出，所以直接就排除了这几个。

  
然后就从所有检测的共同点入手了，所有检测的共同点就是字符串检测，而字符串匹配最常见的就是strstr函数。

  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FCSdur00CuDWLZqs5LNhfO2EExS9xmVMutt8kTRQJibh5qLiad4KI5ibYWHBicN9Cja8KbYQFzwqa7YQ/640?wx_fmt=png&from=appmsg&wxfrom=13)  
  

然后就hook掉修改返回值，之后就尴尬了。

  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FCSdur00CuDWLZqs5LNhfOjbZIzo2kS3zxgHhNBMxfV8N1iaqryEJgCetrwnpb3zcMH2zwGzE23bg/640?wx_fmt=png&from=appmsg&wxfrom=13)  

  

当然还有一些其它的检测，比如里面还对.dex进行了比较，不过目的是处理frida挂不上去的问题，所以就没有细看比较这东西是干嘛的，盲猜可能是检测dex注入的。

  
上面的几个检测点直接用frida进行hook就行了：

  

var ph = Module.findExportByName(null, "strstr");  
    Interceptor.attach(ptr(ph), {  
        onEnter: function (args) {  
            this.filename = args[0]  
            this.checkname = args[1]  
            // console.log(this.filename.readCString(), ptr(args[1]).readCString())  
        }, onLeave: function (retval) {  
  
            var s = ptr(this.filename).readCString()  
            if (s.indexOf("gmain") >= 0){  
                console.log("gmain anti.")  
                retval.replace(0)  
            }else if(s.indexOf("gum-js-loop") >= 0){  
                console.log("gum-js-loop anti.")  
                retval.replace(0)  
            }else if(s.indexOf("linjector") >= 0){  
                console.log("linjector anti.")  
                retval.replace(0)  
            }else if(s.indexOf("/data/local/tmp") >= 0){  
                console.log("/data/local/tmp anti.")  
                retval.replace(0)  
            }  
        }  
    })  

###   

###   

  

三  

  

****定位检测代码位置****

  

定位检测代码的位置还是简单的采用了打印栈的方式：

  

console.log(Thread.backtrace(this.context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join("\n"));  

  
![[Pasted image 20240920134927.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

结合maps文件就直接定位到libmsaoaidsec.so里面的0x1AEE4方法了，里面一堆的字符串解密函数不用管。

  ![[Pasted image 20240920134933.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

想看每一个代码段都是什么字符串的话可以跑一下脚本自解密即可，或者直接动态打印，不过这里没有必要。

  
排除这些解密字符串的代码直接找到里面的函数调用的地方：

  ![[Pasted image 20240920134940.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  

字符串解密后死循环每隔4秒检测一次，初略看了一下，大概是这么个情况，因为没有验证所以仅供参考。

  

sub_1A940 线程名检测  
sub_1AAEC 通过/proc/self/fd文件的实际路径检测是否存在匹配的文件  
sub_1AC00 maps检测  
sub_23804 xxx内容有点大，需要具体分析  

  

sub_23804初看貌似是对libart的某些函数的hook检测，有兴趣的可以分析看看，代码不适合直接stalker进行处理，因为里面像这种指令会导致死循环需要特殊处理，在系统的库代码里面这种原子指令到处都是。

  ![[Pasted image 20240920134950.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

再则，里面好像用到了这个：

  

    v2 = dlopen("libart.so", 0);  
    if ( !v2 )  
      goto LABEL_23;  
    v3 = v2;  
  
    ...  
  
    off_47728[0] = (int *)dlsym(v3, (const char *)&v38);  
    dlclose(v3);  
    goto LABEL_23;  
  
    ...  
  
LABEL_23:  
    v0 = off_47728[0];  
    if ( off_47728[0] )  
      return (unsigned int)*v0 >> 1 == 0x2C000028;  
    return 0LL;

  

参考：

  ![[Pasted image 20240920134959.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

###   

  

四  

  

****尝试bpf定位****

  

### **接下来假设前面的检测手段都被修正了，那么前面的都得失效，比如：**  

  

strstr函数自定义  
相应的方法采用svc的方式调用  
...  

  

这时候使用bpf检直就是降维打击，主要用到了bcc里面的opensnoop及trace.py脚本。

  

python3 opensnoop -u 10094   
python3 trace.py --uid 10094 'do_sys_openat2 "%s", arg2@user' -U --address -v -f maps  

  

opensnoop观察访问的文件日志 ，trace查看堆栈。

  
比如，这里是对这个app进行进行文件追踪的日志：

  ![[Pasted image 20240920135012.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

你会频繁的看到这类日志，而循环读取这类文件的，那最有可能就是检测点了  
既然找到了切入点，那自然是通过trace的栈信息来定位了，trace的用户栈信息有几个问题，一个是栈信息不全，其次就是总是显示unknow的问题，这个有兴趣研究的可以参考大佬的文章_https://bbs.kanxue.com/thread-274546.htm。_

  
而我采用的是最笨最简单的一种方式，就是直接解析maps，将文件偏移与文件名关联起来直接替换trace脚本中的相关内容就行了，虽然没有解决特殊情况下栈不全的问题，但也勉强能用了，毕竟我的要求不高。

  

# 定义  
class MapsItem:  
    def __init__(self, startAddr, endAddr, fOffset, path):  
        self.startAddr = int(startAddr, 16)  
        self.endAddr = int(endAddr, 16)  
        self.fileOffset = int(fOffset, 16)  
        self.path = path  
  
    def update(self, startAddr=None, endAddr=None, path=None):  
        if startAddr is not None:  
            self.startAddr = startAddr  
        if endAddr is not None:  
            self.endAddr = endAddr  
        if path is not None:  
            self.path = path  
  
    def __str__(self):  
        return "MapsItem{%s-%s %s %s}" % (hex(self.startAddr), hex(self.endAddr), hex(self.fileOffset), self.path)  
  
class PidItem(object):  
    pid = 0  
    lst_segments = []     # MapsItem  
  
    def __init__(self, pid):  
        self.pid = pid  
  
    def printLst(self):  
        for item in self.lst_segments:  
            print("%s" % (item))  
  
class KKStackCache:  
    pidmap = {}  
  
    def __init__(self, pid):  
        if pid in KKStackCache.pidmap:  
            pass  
        else:  
            KKStackCache.parserMaps(pid)  
  
    @staticmethod  
    def parserMaps(pid):  
        try:  
            fname = "/proc/%d/maps" % (pid)  
            # fname = "maps"  
            # print("open %s" % fname)  
            f = open(fname)  
            pattern = re.compile(r'([\w]+)-([\w]+)\s+([\w\-]+)\s+([\w\-]+)\s+([\w\:]+)\s+([\d]+)\s+(\S+(?:\s+\S+)*?)*\s*$')  
  
            KKStackCache.pidmap[pid] = PidItem(pid)  
  
            mapsdata = []  
            lines = f.readlines()  
            for line in lines:  
                match = re.match(pattern, line)  
                if match:  
                    mapsItem = MapsItem(match.group(1), match.group(2), match.group(4), match.group(7))  
                    mapsdata.append(mapsItem)  
  
                else:  
                    print("No match:" + line)  
  
            KKStackCache.pidmap[pid].lst_segments = sorted(mapsdata, key=lambda x: x.startAddr)  
  
            f.close()  
        except:  
            pass  
  
        print("parser %d over." % pid)  
  
    @staticmethod  
    def addrInfo(pid, addr):  
        pidmap = KKStackCache.pidmap.get(pid)  
        if pidmap is not None:  
            lst = pidmap.lst_segments  
  
            bg = 0;  
            end = len(lst) - 1  
  
            while bg <= end:  
                mid = int((bg + end + 1) / 2)  
                # print("mid %d: %s %s, addr:%s" % (mid, hex(lst[mid].startAddr), hex(lst[mid].endAddr), hex(addr)))  
                if addr < lst[mid].startAddr:  
                    end = mid - 1  
                elif addr >= lst[mid].endAddr:  
                    bg = mid + 1  
                else:  
                    itm = lst[mid]  
                    path = itm.path  
                    offset = itm.fileOffset + (addr - itm.startAddr)  
                    return True, addr, offset, path  
        return False, None, None, None  
  
    @staticmethod  
    def printLst(pid):  
        pidmap = KKStackCache.pidmap.get(pid)  
        if pidmap is not None:  
            pidmap.printLst()  
# 使用  
# 给_stack_to_string函数增加一个pid参数 def _stack_to_string(self, bpf, stack_id, tgid, pid):  
# 在处理栈信息的地方加上下面的代码  
                if pid > 0:  
                    name, offset, module = BPF._sym_cache(tgid).resolve(addr, False)  
                    if name is None:  
                        KKStackCache(pid)  
                        isExist, baseAddr, offset, path = KKStackCache.addrInfo(pid, addr)  
                        if not isExist:  
                            KKStackCache.parserMaps(pid)  
                            isExist, baseAddr, offset, path = KKStackCache.addrInfo(pid, addr)  
                        if isExist:  
                            if path is not None:  
                                symstr = "%s %s" % (hex(offset), path)  

  

效果如下：

  
![[Pasted image 20240920135030.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  
然后定位的0x1ac50是哪呢，正好是我们前面分析的0x1AC00函数内部，结合我们前面看过的逻辑，不难看出来这栈信息是不全的，不过其实也无所位了，定位到了一个点，其它的也就不远了，比如就像我们之前的操作，直接hook打印堆栈信息等，或者按大佬的方式去处理也可以。

  
至于那个libtiny.so的栈信息，我就打开简单看了下：

  ![[Pasted image 20240920135037.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)  
  

上下翻了翻，能大致看到有读取maps并手动解析的操作，其它略过~（ps:因为不调试细节看不懂，还有就是懒。

  

然后试了一下通过bpf修改open函数的返回值，基于opensnoop.py做的尝试性修改：

  

//在trace_return里面增加代码  
 if(my_strnstr((const char*)data.name, "maps", 256, 5) != NULL){  
        bpf_override_return(ctx, -1);  
    }  

  

其中my_strnstr是自定义实现的strstr函数，因为标准的strstr函数貌似会因为循环问题而过不了验证，所以实现了一个有限循环的strstr，再则，使用bpf_override_return需要内核开启CONFIG_BPF_KPROBE_OVERRIDE选项  
开启和关闭效果如下：  

  
![[Pasted image 20240920135047.png]]
![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

  

bpf很好很强大，强大到颠覆性的改变，可以类比frida对安全行业的影响，而且是完全是高维打击。

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

**看雪ID：lxcoder**

https://bbs.kanxue.com/user-home-719948.htm

*本文为看雪论坛优秀文章，由 lxcoder 原创，转载请注明来自看雪社区

  

[![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545804&idx=3&sn=bb6158b424fc9a25a314891ba45e7098&chksm=b18d5e0686fad7109d8856786cfbe4181b625ae3021dea954b77d17fc8e6d3437a758a2a8357&scene=21#wechat_redirect)

  

  

**#** **往期推荐**

1、[【NKCTF】babyHeap-Off by one&Tcache Attack](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545943&idx=1&sn=26a6b798597e8adb9a1c990e6108601c&chksm=b18d5f9d86fad68b36cc01e3f47e17293361d53495e526e70a4575fbcf0537aa3773fb9bb552&scene=21#wechat_redirect)

2、[Qemu源码浅析之v0.1.6](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545874&idx=1&sn=1bda818b278c74e7e272ccbc5c60e741&chksm=b18d5e5886fad74e3a7e27567485411f5b4082434f18879ef0b34eabb302f77f9deccd2d6b3a&scene=21#wechat_redirect)

3、[堆利用学习：the house of einherjar](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545804&idx=1&sn=1cf30cee29dfca07f63bc47bc38dbafa&chksm=b18d5e0686fad710b1b521066e50869e623ffbed45489c0a965e9f070ae1b9c6690faeb984a2&scene=21#wechat_redirect)

4、[Bugku CTF安卓逆向LoopAndLoop](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545766&idx=1&sn=c5d4c769573620bc28eaae9100f2dd85&chksm=b18d5eec86fad7fa1c261ad8e810b81dbf26904351c4b371ef3231f011c7e5eebbc23d06dc06&scene=21#wechat_redirect)

5、[CVE-2021-32760漏洞分析与复现](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545484&idx=1&sn=ba54c0bae5a9c91ff71d0b350478c90e&chksm=b18d5dc686fad4d02e9f1e23bda7e71a5a989d9d336764ef055bc4a9bf6ae6883f4154bf4a2f&scene=21#wechat_redirect)

6、[Hikari源码分析 - AntiClassDump](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458545483&idx=1&sn=f563e7df11392e42521c3707655680c2&chksm=b18d5dc186fad4d72290c1a14e989815573b3287d52b134c6a5b5e5edd0c9d720425cc158baf&scene=21#wechat_redirect)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球分享**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球点赞**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**球在看**

  

  

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

点击阅读原文查看更多

阅读原文

阅读 2959

​

写留言

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/1UG7KPNHN8EGLfh77kFmnicd9WOic2ibvhCibFdB4bL4srJCgo2wnvdoXLxpIvAkfCmmcptXZB0qKWMoIP8iaibYN2FA/300?wx_fmt=png&wxfrom=18)

看雪学苑

121225

写留言

写留言

**留言**

暂无留言