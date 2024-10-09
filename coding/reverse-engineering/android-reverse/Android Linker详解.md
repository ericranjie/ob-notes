# 

- [Android Linker详解](https://bbs.kanxue.com/thread-269801.htm#android-linker%E8%AF%A6%E8%A7%A3)
  - [本文目的](https://bbs.kanxue.com/thread-269801.htm#%E6%9C%AC%E6%96%87%E7%9B%AE%E7%9A%84)
  - [Linker入口](https://bbs.kanxue.com/thread-269801.htm#linker%E5%85%A5%E5%8F%A3)
  - [So的装载](https://bbs.kanxue.com/thread-269801.htm#so%E7%9A%84%E8%A3%85%E8%BD%BD)
  - [总结](https://bbs.kanxue.com/thread-269801.htm#%E6%80%BB%E7%BB%93)

## [](https://bbs.kanxue.com/thread-269801.htm#%E6%9C%AC%E6%96%87%E7%9B%AE%E7%9A%84)本文目的

Unidbg在对So进行模拟执行的时候，需要先将So文件加载到内存，配置So的进程映像，然后使用CPU模拟器(Unicorn、Dynamic等)对So进行模拟执行。本文的目的是为了彻底搞懂So文件是如何加载到内存的，以及加载进内存之后做了什么，史无巨细，握住方向盘

## [](https://bbs.kanxue.com/thread-269801.htm#linker%E5%85%A5%E5%8F%A3)Linker入口

我们在Android程序中，往往会使用到JNI编程来加快某些算法的运行或增加APP的逆向难度。当然这不是Android的新特性，它是Java自带的本地编程接口，可以使我们的Java程序能够调用本地语言。

当我们在Android程序想使用本地编译的So库，第一步就是要将So加载进来对吧，Android Studio创建C/C++ Native模板的时候，它会在我们的MainActivity类中加这么一段代码

|   |   |
|---|---|
|1<br><br>2<br><br>3|`static{`<br><br>    ``` System.loadLibrary(``"native-lib"``); ```<br><br>`}`|

这句代码的作用就是将So加载进来供Android程序来使用，所以以此为入口，开始分析

> http://androidxref.com/4.4.4_r1/xref/libcore/luni/src/main/java/java/lang/System.java#525

|   |   |
|---|---|
|1<br><br>2<br><br>3|`public static void loadLibrary(String libName) {`<br><br>    `Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());`<br><br>`}`|

又调用了Runtime类的loadLibrary，第二个参数为调用类的ClassLoader

> http://androidxref.com/4.4.4_r1/xref/libcore/luni/src/main/java/java/lang/Runtime.java#354

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37|`void loadLibrary(String libraryName, ClassLoader loader) {`<br><br>    `if` ``` (loader !``= ``` `null) {`<br><br>        ``` /``/``调用findLibrary通过名称(``"native-lib"``)寻找真实库文件名 ```<br><br>        `String filename` `=` `loader.findLibrary(libraryName);`<br><br>        `if` `(filename` ``` =``= ``` `null) {`<br><br>            ``` throw new UnsatisfiedLinkError(``"..."``); ```<br><br>        `}`<br><br>        ``` /``/``如果找到了，进行加载 ```<br><br>        `String error` `=` `doLoad(filename, loader);`<br><br>        `if` ``` (error !``= ``` `null) {`<br><br>            `throw new UnsatisfiedLinkError(error);`<br><br>        `}`<br><br>        ``` return``; ```<br><br>    `}`<br><br>    `String filename` `=` `System.mapLibraryName(libraryName);`<br><br>    ``` List``<String> candidates ``` `=` `new ArrayList<String>();`<br><br>    `String lastError` `=` `null;`<br><br>    `for` `(String directory : mLibPaths) {`<br><br>        `String candidate` `=` `directory` `+` `filename;`<br><br>        `candidates.add(candidate);`<br><br>        `if` `(IoUtils.canOpenReadOnly(candidate)) {`<br><br>            ``` /``/``这里是还有其他的路径来搜索我们的库文件名，都是调用doLoad方法 ```<br><br>            `String error` `=` `doLoad(candidate, loader);`<br><br>            `if` `(error` ``` =``= ``` `null) {`<br><br>                ``` return``; ``` ``` /``/ ``` `We successfully loaded the library. Job done.`<br><br>            `}`<br><br>            `lastError` `=` `error;`<br><br>        `}`<br><br>    `}`<br><br>    `if` ``` (lastError !``= ``` `null) {`<br><br>        `throw new UnsatisfiedLinkError(lastError);`<br><br>    `}`<br><br>    ``` throw new UnsatisfiedLinkError(``"Library " ``` `+` `libraryName` `+` `" not found; tried "` ``` +``candidates); ```<br><br>`}`|

接着看doLoad方法

> http://androidxref.com/4.4.4_r1/xref/libcore/luni/src/main/java/java/lang/Runtime.java#393

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11|`private String doLoad(String name, ClassLoader loader) {`<br><br>    `String ldLibraryPath` `=` `null;`<br><br>    `if` ``` (loader !``= ``` `null && loader instanceof BaseDexClassLoader) {`<br><br>        ``` /``/``如果是BaseDexClassLoader，获取系统so的路径 ```<br><br>        `ldLibraryPath` `=` `((BaseDexClassLoader) loader).getLdLibraryPath();`<br><br>    `}`<br><br>    `synchronized (this) {`<br><br>        ``` /``/``继续调用nativeLoad,还加了同步锁 ```<br><br>        `return` `nativeLoad(name, loader, ldLibraryPath);`<br><br>    `}`<br><br>`}`|

> http://androidxref.com/4.4.4_r1/xref/libcore/luni/src/main/java/java/lang/Runtime.java#426

|   |   |
|---|---|
|1|`private static native String nativeLoad(String filename, ClassLoader loader, String ldLibraryPath);`|

继续往下分析，找到nativeLoad对应的C层函数

> http://androidxref.com/4.4.4_r1/xref/art/runtime/native/java_lang_Runtime.cc#43

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14|``` static jstring Runtime_nativeLoad(JNIEnv``* ``` `env, jclass, jstring javaFilename, jobject javaLoader, jstring javaLdLibraryPath) {`<br><br>  ``` /``/``...各种检查 ```<br><br>  ``` mirror::ClassLoader``* ``` `classLoader` `=` ``` soa.Decode<mirror::ClassLoader``*``>(javaLoader); ```<br><br>  `std::string detail;`<br><br>  ``` JavaVMExt``* ``` `vm` `=` ``` Runtime::Current()``-``>GetJavaVM(); ```<br><br>  `bool` `success` `=` ``` vm``-``>LoadNativeLibrary(filename.c_str(), classLoader, detail); ```<br><br>  `if` `(success) {`<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  ``` /``/ ``` `Don't let a pending exception` `from` `JNI_OnLoad cause a CheckJNI issue with NewStringUTF.`<br><br>  ``` env``-``>ExceptionClear(); ```<br><br>  `return` ``` env``-``>NewStringUTF(detail.c_str()); ```<br><br>`}`|

最关键的函数是vm->LoadNativeLibrary，继续往下跟

> http://androidxref.com/4.4.4_r1/xref/art/runtime/jni_internal.cc#3120

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32|`bool` ``` JavaVMExt::LoadNativeLibrary(const std::string& path, ClassLoader``* ``` `class_loader,`<br><br>                                  `std::string& detail) {`<br><br>    ``` /``/``... ```<br><br>    ``` self``-``>TransitionFromRunnableToSuspended(kWaitingForJniOnLoad); ```<br><br>    ``` /``/``调用dlopen加载So，并返回一个handle句柄 ```<br><br>    ``` void``* ``` `handle` `=` `dlopen(path.empty() ? NULL : path.c_str(), RTLD_LAZY);`<br><br>    ``` self``-``>TransitionFromSuspendedToRunnable(); ```<br><br>    `VLOG(jni) <<` `"[Call to dlopen(\""` `<< path <<` `"\", RTLD_LAZY) returned "` `<< handle <<` ``` "]"``; ```<br><br>    ``` /``/``... ```<br><br>    ``` /``/``此时如果So加载正常，会调用dlsym查找JNI_OnLoad符号，并执行 ```<br><br>    ``` void``* ``` `sym` `=` `dlsym(handle,` ``` "JNI_OnLoad"``); ```<br><br>    `if` `(sym` ``` =``= ``` `NULL) {`<br><br>        `VLOG(jni) <<` `"[No JNI_OnLoad found in \""` `<< path <<` ``` "\"]"``; ```<br><br>        `was_successful` `=` `true;`<br><br>    `}` `else` `{`<br><br>    `}`<br><br>    `typedef` `int` ``` (``*``JNI_OnLoadFn)(JavaVM``*``, void``*``); ```<br><br>    `JNI_OnLoadFn jni_on_load` `=` `reinterpret_cast<JNI_OnLoadFn>(sym);`<br><br>    ``` ClassLoader``* ``` `old_class_loader` `=` ``` self``-``>GetClassLoaderOverride(); ```<br><br>    ``` self``-``>SetClassLoaderOverride(class_loader); ```<br><br>    `int` `version` `=` ``` 0``; ```<br><br>    `{`<br><br>        ``` ScopedThreadStateChange tsc(``self``, kNative); ```<br><br>        `VLOG(jni) <<` `"[Calling JNI_OnLoad in \""` `<< path <<` ``` "\"]"``; ```<br><br>        ``` /``/``在这里调用JNIOnload ```<br><br>        `version` `=` ``` (``*``jni_on_load)(this, NULL); ```<br><br>    `}`<br><br>    ``` /``/``... ```<br><br>    ``` library``-``>SetResult(was_successful); ```<br><br>    `return` `was_successful;`<br><br>`}`|

我们分析上面的函数知道，此函数主要做了两件事

- 调用dlopen加载So
- 查找So中的JNI_OnLoad函数，并执行\
  继续往下分析dlopen

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/dlfcn.cpp#63

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9|``` void``* ``` ``` dlopen(const char``* ``` `filename,` `int` `flags) {`<br><br>  `ScopedPthreadMutexLocker locker(&gDlMutex);`<br><br>  ``` soinfo``* ``` `result` `=` `do_dlopen(filename, flags);`<br><br>  `if` `(result` ``` =``= ``` `NULL) {`<br><br>    ``` __bionic_format_dlerror(``"dlopen failed"``, linker_get_error_buffer()); ```<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  `return` `result;`<br><br>`}`|

调用了do_dlopen

## [](https://bbs.kanxue.com/thread-269801.htm#so%E7%9A%84%E8%A3%85%E8%BD%BD)So的装载

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#823

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13|``` soinfo``* ``` ``` do_dlopen(const char``* ``` `name,` `int` `flags) {`<br><br>  `if` ``` ((flags & ~(RTLD_NOW\|RTLD_LAZY\|RTLD_LOCAL\|RTLD_GLOBAL)) !``= ``` ``` 0``) { ```<br><br>    ``` DL_ERR(``"invalid flags to dlopen: %x"``, flags); ```<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  `set_soinfo_pool_protection(PROT_READ \| PROT_WRITE);`<br><br>  ``` soinfo``* ``` `si` `=` `find_library(name);`<br><br>  `if` ``` (si !``= ``` `NULL) {`<br><br>    ``` si``-``>CallConstructors(); ```<br><br>  `}`<br><br>  `set_soinfo_pool_protection(PROT_READ);`<br><br>  `return` `si;`<br><br>`}`|

分析到这里，终于进入Linker部分了，上面的篇幅我们由System.loadLibrary()方法，找到了Linker的do_dlopen函数，这个函数就可以说是Linker开始加载的地方了。这个函数主要做了两件事

- 调用函数find_library，返回soinfo。soinfo就是so被加载到内存的一个代表，存放了内存中so的信息
- 调用soinfo的CallConstructors函数，做了一些初始化操作(Iint、init.array)

继续分析find_library

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#785

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7|``` static soinfo``* ``` ``` find_library(const char``* ``` `name) {`<br><br>  ``` soinfo``* ``` `si` `=` `find_library_internal(name);`<br><br>  `if` ``` (si !``= ``` `NULL) {`<br><br>    ``` si``-``>ref_count``+``+``; ```<br><br>  `}`<br><br>  `return` `si;`<br><br>`}`|

这个函数的作用很简单

- 调用find_library_internal
- so的引用计数+1

继续分析find_library_internal函数

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#751

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34|``` static soinfo``* ``` ``` find_library_internal(const char``* ``` `name) {`<br><br>  `if` `(name` ``` =``= ``` `NULL) {`<br><br>    `return` `somain;`<br><br>  `}`<br><br>  ``` /``/``寻找已经加载过的so，当我们的so被加载完成后，会放到已加载列表，再次调用System.loadLibrary的时候不需要进行二次加载 ```<br><br>  ``` soinfo``* ``` `si` `=` `find_loaded_library(name);`<br><br>  `if` ``` (si !``= ``` `NULL) {`<br><br>    `if` ``` (si``-``>flags & FLAG_LINKED) { ```<br><br>      `return` `si;`<br><br>    `}`<br><br>    ``` DL_ERR(``"OOPS: recursive link to \"%s\""``, si``-``>name); ```<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  ``` TRACE(``"[ '%s' has not been loaded yet.  Locating...]"``, name); ```<br><br>  ``` /``/``如果没有被加载过，就调用load_library进行加载 ```<br><br>  `si` `=` `load_library(name);`<br><br>  `if` `(si` ``` =``= ``` `NULL) {`<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  ``` /``/ ``` `At this point we know that whatever` `is` `loaded @ base` `is` `a valid ELF`<br><br>  ``` /``/ ``` `shared library whose segments are properly mapped` ``` in``. ```<br><br>  ``` TRACE(``"[ init_library base=0x%08x sz=0x%08x name='%s' ]"``, ```<br><br>        ``` si``-``>base, si``-``>size, si``-``>name); ```<br><br>  ``` /``/ ``` `so被加载后，进行链接`<br><br>  `if` `(!soinfo_link_image(si)) {`<br><br>    ``` munmap(reinterpret_cast<void``*``>(si``-``>base), si``-``>size); ```<br><br>    `soinfo_free(si);`<br><br>    `return` `NULL;`<br><br>  `}`<br><br>  `return` `si;`<br><br>`}`|

这个函数主要做了3个事情：

- 判断想要加载的so是否已经被加载过
- 如果没有被加载过，调用load_library进行加载
- 加载完成后，调用soinfo_link_image函数进行链接\
  也就体现了我们So装载的主要两个步骤
- So的装载
- So的链接\
  在上面我们还有一个调用soinfo的CallConstructors函数，这个也可以作为第三个
- So的初始化

那么我们假设我们的So是第一次进行加载，继续分析load_library函数，看看linker如何装载我们的So

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker.cpp#702

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29|``` static soinfo``* ``` ``` load_library(const char``* ``` `name) {`<br><br>    ``` /``/ ``` `打开So文件，拿到文件描述符fd`<br><br>    `int` `fd` `=` `open_library(name);`<br><br>    `if` `(fd` ``` =``= ``` ``` -``1``) { ```<br><br>        ``` DL_ERR(``"library \"%s\" not found"``, name); ```<br><br>        `return` `NULL;`<br><br>    `}`<br><br>    ``` /``/``创建ElfReader对象，并调用Load方法 ```<br><br>    `ElfReader elf_reader(name, fd);`<br><br>    `if` `(!elf_reader.Load()) {`<br><br>        `return` `NULL;`<br><br>    `}`<br><br>    ``` /``/``生成soinfo，并根据elf_reader的结果进行赋值 ```<br><br>    ``` const char``* ``` `bname` `=` `strrchr(name,` ``` '/'``); ```<br><br>    ``` soinfo``* ``` `si` `=` `soinfo_alloc(bname ? bname` `+` `1` `: name);`<br><br>    `if` `(si` ``` =``= ``` `NULL) {`<br><br>        `return` `NULL;`<br><br>    `}`<br><br>    ``` si``-``>base ``` `=` `elf_reader.load_start();`<br><br>    ``` si``-``>size ``` `=` `elf_reader.load_size();`<br><br>    ``` si``-``>load_bias ``` `=` `elf_reader.load_bias();`<br><br>    ``` si``-``>flags ``` `=` ``` 0``; ```<br><br>    ``` si``-``>entry ``` `=` ``` 0``; ```<br><br>    ``` si``-``>dynamic ``` `=` `NULL;`<br><br>    ``` si``-``>phnum ``` `=` `elf_reader.phdr_count();`<br><br>    ``` si``-``>phdr ``` `=` `elf_reader.loaded_phdr();`<br><br>    `return` `si;`<br><br>`}`|

那么我们主要来分析elf_reader.Load()函数

> http://androidxref.com/4.4.4_r1/xref/bionic/linker/linker_phdr.cpp#134

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8|`bool` `ElfReader::Load() {`<br><br>  `return` `ReadElfHeader() &&`<br><br>         `VerifyElfHeader() &&`<br><br>         `ReadProgramHeader() &&`<br><br>         `ReserveAddressSpace() &&`<br><br>         `LoadSegments() &&`<br><br>         `FindPhdr();`<br><br>`}`|

Load函数分别调用了6个函数

- ReadElfHeader 读取ElfHeader
- VerifyElfHeader 验证ElfHeader
- ReadProgramHeader 读取程序头表
- ReserveAddressSpace 准备地址空间
- LoadSegments 加载段
- FindPhdr 寻找Phdr段

从函数名直译，我们也能知道一个大概。下面我们来分析这6个函数

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13|`bool` `ElfReader::ReadElfHeader() {`<br><br>  ``` /``/``从我们打开的So文件中，读取header长度的内容赋值到header_ ```<br><br>  `ssize_t rc` `=` `TEMP_FAILURE_RETRY(read(fd_, &header_, sizeof(header_)));`<br><br>  `if` `(rc <` ``` 0``) { ```<br><br>    ``` DL_ERR(``"can't read file \"%s\": %s"``, name_, strerror(errno)); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  `if` ``` (rc !``= ``` `sizeof(header_)) {`<br><br>    ``` DL_ERR(``"\"%s\" is too small to be an ELF executable"``, name_); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  `return` `true;`<br><br>`}`|

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42<br><br>43<br><br>44|`bool` `ElfReader::VerifyElfHeader() {`<br><br>  ``` /``/``校验魔数 ```<br><br>  `if` ``` (header_.e_ident[EI_MAG0] !``= ``` `ELFMAG0 \|`<br><br>      ``` header_.e_ident[EI_MAG1] !``= ``` `ELFMAG1 \|`<br><br>      ``` header_.e_ident[EI_MAG2] !``= ``` `ELFMAG2 \|`<br><br>      ``` header_.e_ident[EI_MAG3] !``= ``` `ELFMAG3) {`<br><br>    ``` DL_ERR(``"\"%s\" has bad ELF magic"``, name_); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/``因为分析的是Android4.``4``源码，所以它必须是一个``32``位的So ```<br><br>  `if` ``` (header_.e_ident[EI_CLASS] !``= ``` `ELFCLASS32) {`<br><br>    ``` DL_ERR(``"\"%s\" not 32-bit: %d"``, name_, header_.e_ident[EI_CLASS]); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/``必须为小端字节序 ```<br><br>  `if` ``` (header_.e_ident[EI_DATA] !``= ``` `ELFDATA2LSB) {`<br><br>    ``` DL_ERR(``"\"%s\" not little-endian: %d"``, name_, header_.e_ident[EI_DATA]); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/``必须为ET_DYN，也就是我们的Shared ``` `Object` `So文件`<br><br>  `if` ``` (header_.e_type !``= ``` `ET_DYN) {`<br><br>    ``` DL_ERR(``"\"%s\" has unexpected e_type: %d"``, name_, header_.e_type); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/ ``` ``` version当前版本，这个一般都是EV_CURRENT(``1``) ```<br><br>  `if` ``` (header_.e_version !``= ``` `EV_CURRENT) {`<br><br>    ``` DL_ERR(``"\"%s\" has unexpected e_version: %d"``, name_, header_.e_version); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/ ``` `校验e_machine`<br><br>  `if` ``` (header_.e_machine !``= ```<br><br>`#ifdef ANDROID_ARM_LINKER`<br><br>      `EM_ARM`<br><br>`#elif defined(ANDROID_MIPS_LINKER)`<br><br>      `EM_MIPS`<br><br>`#elif defined(ANDROID_X86_LINKER)`<br><br>      `EM_386`<br><br>`#endif`<br><br>  `) {`<br><br>    ``` DL_ERR(``"\"%s\" has unexpected e_machine: %d"``, name_, header_.e_machine); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  `return` `true;`<br><br>`}`|

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30|`bool` `ElfReader::ReadProgramHeader() {`<br><br>  ``` /``/``e_phnum 为我们So的程序头表(段)数，后面我们都叫段，是一个意思 ```<br><br>  `phdr_num_` `=` `header_.e_phnum;`<br><br>  ``` /``/ ``` `Like the kernel, we only accept program header tables that`<br><br>  ``` /``/ ``` `are smaller than` ``` 64KiB``. ```<br><br>  `if` `(phdr_num_ <` `1` `\| phdr_num_ >` ``` 65536``/``sizeof(Elf32_Phdr)) { ```<br><br>    ``` DL_ERR(``"\"%s\" has invalid e_phnum: %d"``, name_, phdr_num_); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` /``/``e_phoff 为我们段表在文件中的偏移, 然后进行内存页对其 ```<br><br>  `Elf32_Addr page_min` `=` `PAGE_START(header_.e_phoff);`<br><br>  `Elf32_Addr page_max` `=` `PAGE_END(header_.e_phoff` `+` `(phdr_num_` `*` `sizeof(Elf32_Phdr)));`<br><br>  `Elf32_Addr page_offset` `=` `PAGE_OFFSET(header_.e_phoff);`<br><br>  ``` /``/ ``` `在Linux中内存读写都是以页为单位，所以上面按照存放段的位置，算出了需要的页大小`<br><br>  ``` /``/ ``` `page_offset就是段在该页的一个偏移`<br><br>  `phdr_size_` `=` `page_max` `-` `page_min;`<br><br>  ``` /``/``将该包含段表的页映射到内存 ```<br><br>  ``` void``* ``` `mmap_result` `=` `mmap(NULL, phdr_size_, PROT_READ, MAP_PRIVATE, fd_, page_min);`<br><br>  `if` `(mmap_result` ``` =``= ``` `MAP_FAILED) {`<br><br>    ``` DL_ERR(``"\"%s\" phdr mmap failed: %s"``, name_, strerror(errno)); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  `phdr_mmap_` `=` `mmap_result;`<br><br>  ``` /``/ ``` `phdr_table_ 就指向了段表在内存中的起始位置`<br><br>  `phdr_table_` `=` ``` reinterpret_cast<Elf32_Phdr``*``>(reinterpret_cast<char``*``>(mmap_result) ``` ``` +``page_offset); ```<br><br>  `return` `true;`<br><br>`}`|

|                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35 | `bool` `ElfReader::ReserveAddressSpace() {`<br><br>  ``` /``/``此时段表已经被加载到内存 ```<br><br>  `Elf32_Addr min_vaddr;`<br><br>  ``` /``/``先获取该So的load_size，也就是需要加载的大小，先看下面对该函数的解释 ```<br><br>  `load_size_` `=` `phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);`<br><br>  `if` `(load_size_` ``` =``= ``` ``` 0``) { ```<br><br>    ``` DL_ERR(``"\"%s\" has no loadable segments"``, name_); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  ``` uint8_t``* ``` `addr` `=` ``` reinterpret_cast<uint8_t``*``>(min_vaddr); ```<br><br>  `int` `mmap_flags` `=` `MAP_PRIVATE \| MAP_ANONYMOUS;`<br><br>  ``` /``/``根据PT_LOAD段的指示，匿名映射一块足够装下我们So的内存 ```<br><br>  ``` void``* ``` `start` `=` `mmap(addr, load_size_, PROT_NONE, mmap_flags,` ``` -``1``, ``` ``` 0``); ```<br><br>  `if` `(start` ``` =``= ``` `MAP_FAILED) {`<br><br>    ``` DL_ERR(``"couldn't reserve %d bytes of address space for \"%s\""``, load_size_, name_); ```<br><br>    `return` `false;`<br><br>  `}`<br><br>  `load_start_` `=` `start;`<br><br>  ``` /``/``这里需要注意一下，start为我们映射出来的那块内存的起始地址，按理说它就是我们So加载的一个起始地址 ```<br><br>  ``` /``/``那这里又计算了一个load_bias_是什么意思呢？ ```<br><br>  ``` /``/``So文件并没有对p_vaddr有特殊要求，所以它可以是任意地址，如果它指定了一个最小的虚拟地址不为``0 ```<br><br>  ``` /``/``那么文件中的关于地址的引用就是根据它指定的虚拟地址来的 ```<br><br>  ``` /``/``所以我们在后面进行对地址修正的时候，就要计算 start ``` `-` `min_addr来得到正确的值`<br><br>  ``` /``/``所以这里计算了load_bias_， 后面关于地址引用的地方，我们都用这个load_bias_就可以了 ```<br><br>  ``` /``/``举个例子：假设一个So中的PT_LOAD段指定的最小虚拟地址min_vaddr ``` `=` `0x100`<br><br>  ``` /``/``那么如果这个So中的一个函数中引用了一个地址为``0x300``地方的字符串 ```<br><br>  ``` /``/``那这个字符串在实际文件中的偏移就是``0x200 ``` `=` `0x300` `-` `0x100`<br><br>  ``` /``/``当So加载到内存中，需要对这个函数中的引用做重定位的时候，就应该这样计算 ```<br><br>  ``` /``/``start ``` `+` `0x300` `-` `0x100` ``` <``=``=``> start ``` `-` `0x100`  `+` `0x300`<br><br>  ``` /``/``每次在计算的时候都要``-``0x100``，所以这里就计算了一个load_bias_ ``` `=` `start` `-` `0x100`<br><br>  ``` /``/``后面直接用这个load_bias_ ``` `+` ``` 0x300``(地址引用偏移) 就可以了 ```<br><br>  `load_bias_` `=` ``` reinterpret_cast<uint8_t``*``>(start) ``` `-` `addr;`<br><br>  `return` `true;`<br><br>`}` |

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42|``` size_t phdr_table_get_load_size(const Elf32_Phdr``* ``` `phdr_table,`<br><br>                                `size_t phdr_count,`<br><br>                                ``` Elf32_Addr``* ``` `out_min_vaddr,`<br><br>                                ``` Elf32_Addr``* ``` `out_max_vaddr)`<br><br>`{`<br><br>    `Elf32_Addr min_vaddr` `=` ``` 0xFFFFFFFFU``; ```<br><br>    `Elf32_Addr max_vaddr` `=` ``` 0x00000000U``; ```<br><br>    `bool` `found_pt_load` `=` `false;`<br><br>    ``` /``/``遍历我们的段表 ```<br><br>    `for` `(size_t i` `=` ``` 0``; i < phdr_count; ``` ``` +``+``i) { ```<br><br>        ``` const Elf32_Phdr``* ``` `phdr` `=` `&phdr_table[i];`<br><br>        ``` /``/``只处理PT_LOAD段，因为PT_LOAD段说明了我们的So应该怎么加载 ```<br><br>        `if` ``` (phdr``-``>p_type !``= ``` `PT_LOAD) {`<br><br>            ``` continue``; ```<br><br>        `}`<br><br>        `found_pt_load` `=` `true;`<br><br>        ``` /``/``遍历所有的PT_LOAD段，寻找So指定的最小的一个虚拟地址 ```<br><br>        `if` ``` (phdr``-``>p_vaddr < min_vaddr) { ```<br><br>            `min_vaddr` `=` ``` phdr``-``>p_vaddr; ```<br><br>        `}`<br><br>        ``` /``/``遍历所有的PT_LOAD段，寻找So指定的要加载到内存的最大的一个虚拟地址 ```<br><br>        `if` ``` (phdr``-``>p_vaddr ``` `+` ``` phdr``-``>p_memsz > max_vaddr) { ```<br><br>            `max_vaddr` `=` ``` phdr``-``>p_vaddr ``` `+` ``` phdr``-``>p_memsz; ```<br><br>        `}`<br><br>    `}`<br><br>    `if` `(!found_pt_load) {`<br><br>        `min_vaddr` `=` ``` 0x00000000U``; ```<br><br>    `}`<br><br>    ``` /``/``So文件并没有对p_vaddr有特殊要求，所以这里需要页对齐 ```<br><br>    `min_vaddr` `=` `PAGE_START(min_vaddr);`<br><br>    `max_vaddr` `=` `PAGE_END(max_vaddr);`<br><br>    `if` ``` (out_min_vaddr !``= ``` `NULL) {`<br><br>        ``` *``out_min_vaddr ``` `=` `min_vaddr;`<br><br>    `}`<br><br>    `if` ``` (out_max_vaddr !``= ``` `NULL) {`<br><br>        ``` *``out_max_vaddr ``` `=` `max_vaddr;`<br><br>    `}`<br><br>    ``` /``/``最大``-``最小拿到该So加载到内存的一个size ```<br><br>    `return` `max_vaddr` `-` `min_vaddr;`<br><br>`}`|

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24<br><br>25<br><br>26<br><br>27<br><br>28<br><br>29<br><br>30<br><br>31<br><br>32<br><br>33<br><br>34<br><br>35<br><br>36<br><br>37<br><br>38<br><br>39<br><br>40<br><br>41<br><br>42<br><br>43<br><br>44<br><br>45<br><br>46<br><br>47<br><br>48<br><br>49<br><br>50<br><br>51<br><br>52<br><br>53<br><br>54<br><br>55<br><br>56<br><br>57<br><br>58<br><br>59<br><br>60<br><br>61<br><br>62<br><br>63<br><br>64|``` /``/``上面ReserveAddressSpace函数，只是开辟了一块足够的内存，并没有内容 ```<br><br>``` /``/``这个函数就在填充内容 ```<br><br>`bool` `ElfReader::LoadSegments() {`<br><br>  `for` `(size_t i` `=` ``` 0``; i < phdr_num_; ``` ``` +``+``i) { ```<br><br>    ``` const Elf32_Phdr``* ``` `phdr` `=` `&phdr_table_[i];`<br><br>    ``` /``/``还是在遍历每一个PT_LOAD段 ```<br><br>    `if` ``` (phdr``-``>p_type !``= ``` `PT_LOAD) {`<br><br>      ``` continue``; ```<br><br>    `}`<br><br>    ``` /``/ ``` `计算该PT_LOAD段在内存中的开始地址和结束地址`<br><br>    `Elf32_Addr seg_start` `=` ``` phdr``-``>p_vaddr ``` `+` `load_bias_;`<br><br>    `Elf32_Addr seg_end`   `=` `seg_start` `+` ``` phdr``-``>p_memsz; ```<br><br>    `Elf32_Addr seg_page_start` `=` `PAGE_START(seg_start);`<br><br>    `Elf32_Addr seg_page_end`   `=` `PAGE_END(seg_end);`<br><br>    ``` /``/``计算该PT_LOAD段在内存中对应文件的结束位置 ```<br><br>    `Elf32_Addr seg_file_end`   `=` `seg_start` `+` ``` phdr``-``>p_filesz; ```<br><br>    ``` /``/``文件中的偏移 ```<br><br>    `Elf32_Addr file_start` `=` ``` phdr``-``>p_offset; ```<br><br>    `Elf32_Addr file_end`   `=` `file_start` `+` ``` phdr``-``>p_filesz; ```<br><br>    `Elf32_Addr file_page_start` `=` `PAGE_START(file_start);`<br><br>    `Elf32_Addr file_length` `=` `file_end` `-` `file_page_start;`<br><br>    `if` ``` (file_length !``= ``` ``` 0``) { ```<br><br>      ``` /``/``将该PT_LOAD段的实际内容页对齐后映射到内存中 ```<br><br>      ``` void``* ``` `seg_addr` `=` ``` mmap((void``*``)seg_page_start, ```<br><br>                            `file_length,`<br><br>                            ``` PFLAGS_TO_PROT(phdr``-``>p_flags), ```<br><br>                            `MAP_FIXED\|MAP_PRIVATE,`<br><br>                            `fd_,`<br><br>                            `file_page_start);`<br><br>      `if` `(seg_addr` ``` =``= ``` `MAP_FAILED) {`<br><br>        ``` DL_ERR(``"couldn't map \"%s\" segment %d: %s"``, name_, i, strerror(errno)); ```<br><br>        `return` `false;`<br><br>      `}`<br><br>    `}`<br><br>    ``` /``/``如果该段的权限可写且该段指定的文件大小并不是页边界对齐的，就要对页内没有文件与之对应的区域置``0 ```<br><br>    `if` ``` ((phdr``-``>p_flags & PF_W) !``= ``` `0` `&& PAGE_OFFSET(seg_file_end) >` ``` 0``) { ```<br><br>      ``` memset((void``*``)seg_file_end, ``` ``` 0``, PAGE_SIZE ``` `-` `PAGE_OFFSET(seg_file_end));`<br><br>    `}`<br><br>    `seg_file_end` `=` `PAGE_END(seg_file_end);`<br><br>    ``` /``/ ``` `如果该段指定的内存大小超出了文件映射的页面，就要对多出的页进行匿名映射`<br><br>    ``` /``/ ``` `防止出现Bus error的情况`<br><br>    `if` `(seg_page_end > seg_file_end) {`<br><br>      ``` void``* ``` `zeromap` `=` ``` mmap((void``*``)seg_file_end, ```<br><br>                           `seg_page_end` `-` `seg_file_end,`<br><br>                           ``` PFLAGS_TO_PROT(phdr``-``>p_flags), ```<br><br>                           `MAP_FIXED\|MAP_ANONYMOUS\|MAP_PRIVATE,`<br><br>                           ``` -``1``, ```<br><br>                           ``` 0``); ```<br><br>      `if` `(zeromap` ``` =``= ``` `MAP_FAILED) {`<br><br>        ``` DL_ERR(``"couldn't zero fill \"%s\" gap: %s"``, name_, strerror(errno)); ```<br><br>        `return` `false;`<br><br>      `}`<br><br>    `}`<br><br>  `}`<br><br>  `return` `true;`<br><br>`}`|

|   |   |
|---|---|
|1<br><br>2<br><br>3<br><br>4<br><br>5<br><br>6<br><br>7<br><br>8<br><br>9<br><br>10<br><br>11<br><br>12<br><br>13<br><br>14<br><br>15<br><br>16<br><br>17<br><br>18<br><br>19<br><br>20<br><br>21<br><br>22<br><br>23<br><br>24|``` /``/``这个函数看看就可以，就是在找PT_PHDR段并检查，此段指定了段表本身的位置和大小 ```<br><br>`bool` `ElfReader::FindPhdr() {`<br><br>  ``` const Elf32_Phdr``* ``` `phdr_limit` `=` `phdr_table_` `+` `phdr_num_;`<br><br>  ``` /``/ ``` `If there` `is` `a PT_PHDR, use it directly.`<br><br>  `for` ``` (const Elf32_Phdr``* ``` `phdr` `=` `phdr_table_; phdr < phdr_limit;` ``` +``+``phdr) { ```<br><br>    `if` ``` (phdr``-``>p_type ``` ``` =``= ``` `PT_PHDR) {`<br><br>      `return` `CheckPhdr(load_bias_` `+` ``` phdr``-``>p_vaddr); ```<br><br>    `}`<br><br>  `}`<br><br>  `for` ``` (const Elf32_Phdr``* ``` `phdr` `=` `phdr_table_; phdr < phdr_limit;` ``` +``+``phdr) { ```<br><br>    `if` ``` (phdr``-``>p_type ``` ``` =``= ``` `PT_LOAD) {`<br><br>      `if` ``` (phdr``-``>p_offset ``` ``` =``= ``` ``` 0``) { ```<br><br>        `Elf32_Addr  elf_addr` `=` `load_bias_` `+` ``` phdr``-``>p_vaddr; ```<br><br>        ``` const Elf32_Ehdr``* ``` `ehdr` `=` ``` (const Elf32_Ehdr``*``)(void``*``)elf_addr; ```<br><br>        `Elf32_Addr  offset` `=` ``` ehdr``-``>e_phoff; ```<br><br>        `return` `CheckPhdr((Elf32_Addr)ehdr` `+` `offset);`<br><br>      `}`<br><br>      ``` break``; ```<br><br>    `}`<br><br>  `}`<br><br>  ``` DL_ERR(``"can't find loaded phdr for \"%s\""``, name_); ```<br><br>  `return` `false;`<br><br>`}`|

至此 So的装载部分就分析完了

## [](https://bbs.kanxue.com/thread-269801.htm#%E6%80%BB%E7%BB%93)总结

总结一下So的装载就是根据So的文件信息，先读入So的头部信息，并进行验证。然后找到段表的位置，遍历段表的每一个段，根据PT_LOAD段指定的信息将So进行装载，如果我们要模拟这个过程，只需要注意一下细节就可以了。相对于So的装载，更难的部分是So的动态链接，我们另起一篇文章来讲解So的动态链接。如果觉得本篇文章对您有用，可以扫码加入我们的星球，不定期分享各种最新的技术
