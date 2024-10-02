今天，猿妹要和大家推荐一个Facebook开源的C++11 组件库——Folly，Folly包含Facebook 广泛使用的各种核心库组件。

Folly是Facebook open source library的缩写，提供了类似 Boost 和 std 库的功能。包括散列、字符串、向量、内存分配、位处理等，满足大规模高性能的需求。

**逻辑设计**

Folly 是一组相对独立的组件，有的简单到几个符号。对内部依赖没有限制，这意味着给定的 folly 模块可以使用任何其他 folly 组件。

所有符号都在顶级命名空间中定义folly，当然宏除外。宏名称为 ALL_UPPERCASE 并且应以FOLLY_. 命名空间folly定义了其他内部命名空间，例如internal或detail。用户代码不应依赖于这些命名空间中的符号。

Folly 也有一个experimental目录。这一名称主要意味着我们认为 API 可能会随着时间的推移发生重大变化。通常，此代码仍在大量使用并且经过良好测试。

**Folly安装下载**

folly 支持 gcc (5.1+)、clang 或 MSVC。它支持在 Linux（x86-32、x86-64 和 ARM）、iOS、macOS 和 Windows (x86-64) 上运行。你可以使用以下命令下载安装：

```bash
wget <https://github.com/google/googletest/archive/release-1.8.0.tar.gz> && \\
tar zxf release-1.8.0.tar.gz && \\
rm -f release-1.8.0.tar.gz && \\
cd googletest-release-1.8.0 && \\
cmake . && \\
make && \\
make install
```

**构建测试**

默认情况下，构建测试作为CMake all目标的一部分是禁用的。要构建测试，请在配置时将-DBUILD_TESTS=ON指定为CMake。

**Ubuntu 16.04 LTS**

需要以下软件包（随意剪切和粘贴下面的 apt-get 命令）：

```bash
sudo apt-get install \\
    g++ \\
    cmake \\
    libboost-all-dev \\
    libevent-dev \\
    libdouble-conversion-dev \\
    libgoogle-glog-dev \\
    libgflags-dev \\
    libiberty-dev \\
    liblz4-dev \\
    liblzma-dev \\
    libsnappy-dev \\
    make \\
    zlib1g-dev \\
    binutils-dev \\
    libjemalloc-dev \\
    libssl-dev \\
    pkg-config \\
    libunwind-dev

```

Folly 依赖需要从源代码安装的fmt。以下命令将下载、编译和安装 fmt。

```bash
git clone <https://github.com/fmtlib/fmt.git> && cd fmt

mkdir _build && cd _build
cmake ..

make -j$(nproc)
sudo make install

```

如果需要高级调试功能，请使用

```bash
sudo apt-get install \\
    libunwind8-dev \\
    libelf-dev \\
    libdwarf-dev

```

在 folly 目录（例如 checkout 根目录或存档解包根目录）中，运行：

```bash
mkdir _build && cd _build
  cmake ..
  make -j $(nproc)
  make install # with either sudo or DESTDIR as necessary

```

[https://mmbiz.qpic.cn/sz_mmbiz_png/kOTNkic5gVBFjRicrn8FqpDahibIs4UO8uC3YLfjpRuuOa22MKSWL2L9kVtXfRZr2D3nA4d1T9ia8weAardtHZHoZA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/sz_mmbiz_png/kOTNkic5gVBFjRicrn8FqpDahibIs4UO8uC3YLfjpRuuOa22MKSWL2L9kVtXfRZr2D3nA4d1T9ia8weAardtHZHoZA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

目前，Folly已经在Github上标星17.6K，累计分支4K（Github地址：[https://github.com/facebook/folly）阅读Folly的代码对C++程序员成长也有很大帮助，希望这个项目你会喜欢。](https://github.com/facebook/folly%EF%BC%89%E9%98%85%E8%AF%BBFolly%E7%9A%84%E4%BB%A3%E7%A0%81%E5%AF%B9C++%E7%A8%8B%E5%BA%8F%E5%91%98%E6%88%90%E9%95%BF%E4%B9%9F%E6%9C%89%E5%BE%88%E5%A4%A7%E5%B8%AE%E5%8A%A9%EF%BC%8C%E5%B8%8C%E6%9C%9B%E8%BF%99%E4%B8%AA%E9%A1%B9%E7%9B%AE%E4%BD%A0%E4%BC%9A%E5%96%9C%E6%AC%A2%E3%80%82)