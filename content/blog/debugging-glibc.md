---
external: false
title: gdb调试如何进入glibc
description: how to debugging inside glibc
date: 2023-08-21
---

在使用gdb进行debug时，有时候我们需要调试glibc(libc)，例如pthread_create, fgets等等，即使我们的二进制在编译阶段已经加了`-g`选项，但是我们单步进入的时候会遇到`No such file or directory`的错误，这是因为我们使用的glibc库函数存在于动态链接的`libc.so.6`中，但我们二进制所链接的glibc是不存在 **调试信息(debug info)** 的. 接下来笔者会一步一步介绍如何生成带debug info的glibc，并通过gdb配置使glibc变得可调试。

## Debug info
我们可以通过`readelf -S /path/to/shared_library | grep debug` 来查看某个库是否包含debug info。  
这里以笔者机器上的glibc为例，首先查看glibc库的路径: `sudo ldconfig -p | grep libc.so.6`

```bash
$ sudo ldconfig -p | grep libc.so.6
	libc.so.6 (libc6,x86-64) => /lib/x86_64-linux-gnu/libc.so.6
	libc.so.6 (libc6) => /lib32/libc.so.6
```

查看glibc库是否包含debug info: 
```bash
$ readelf -S /lib/x86_64-linux-gnu/libc.so.6 | grep debug
  [62] .gnu_debuglink    PROGBITS         0000000000000000  001d3ff4
```

包含以上及其类似结果表示glibc不包含debug info，包含debug info的glibc库的结果如下, section名前缀会包含.debug。
```bash
  [67] .debug_aranges    PROGBITS         0000000000000000  001b5180
  [68] .debug_info       PROGBITS         0000000000000000  001c8760
  [69] .debug_abbrev     PROGBITS         0000000000000000  00b8015b
  [70] .debug_line       PROGBITS         0000000000000000  00c8cd3f
  [71] .debug_str        PROGBITS         0000000000000000  00e700d8
  [72] .debug_loc        PROGBITS         0000000000000000  00e9c233
  [73] .debug_ranges     PROGBITS         0000000000000000  01138457
```

## 源码编译带debug info的glibc
笔者以glibc 2.36版本(笔者机器上glibc的版本)源码为例子, 从[glibc官方下载地址](https://ftp.gnu.org/gnu/glibc/)下载glibc源码并解压
```bash
$ wget https://ftp.gnu.org/gnu/glibc/glibc-2.36.tar.gz && tar -xzf glibc-2.36.tar.gz
$ cd glibc-2.36.tar.gz
```
创建单独的目录用于构建，并配置编译选项。  
```bash
$ mkdir build && cd build
$ ../configure --prefix=/usr/share/glibc-2.36
```

必须创建单独的build目录用于构建，否则会出现如下错误
```bash
configure: error: you must configure in a separate build directory
```
其中`configure`的详细参数配置可以参考[glibc configure](https://www.gnu.org/software/libc/manual/html_node/Configuring-and-compiling.html)，默认不加编译参数`CFLAGS`的话，glibc会指定为`-g -O2`，也就是说默认会生成带有debug info的glibc，`--prefix`参数指定了glibc会被安装到哪个路径，这里只能是绝对路径才有效。

源码编译glibc, 等待编译结束
```bash
$ make -j`nproc`
```
在编译完成后，我们会看到当前目录, 也就是build目录下，已经生成了glibc动态链接库，查看一下已经带有debug info.
```bash
$ readelf -S libc.so | grep debug
  [61] .debug_aranges    PROGBITS         0000000000000000  001c8020
  [62] .debug_info       PROGBITS         0000000000000000  001e0370
  [63] .debug_abbrev     PROGBITS         0000000000000000  00760738
  [64] .debug_line       PROGBITS         0000000000000000  0084da65
  [65] .debug_str        PROGBITS         0000000000000000  0098dc2f
  [66] .debug_line_str   PROGBITS         0000000000000000  009bc7a5
  [67] .debug_loclists   PROGBITS         0000000000000000  009c8035
  [68] .debug_rnglists   PROGBITS         0000000000000000  00b28ceb
```

安装glibc
```bash
$ sudo make install
```
命令执行结束后，glibc会被安装`/usr/share/glibc-2.36`下, 动态链接库会位于`/usr/share/glibc-2.36/lib`目录下

## 调试glibc

### gdbinit配置
我们只需要在.gdbinit中设置`set env LD_LIBRARY_PATH=/path/to/shared_library`就可以调试glibc了。
```bash
$ echo "set env LD_LIBRARY_PATH=/usr/share/glibc-2.36/lib" >> ~/.gdbinit
```

### debugging inside glibc
我们以hello world为例子进行调试，hello world代码如下: 
```c
#include <stdio.h>

int main() {
    printf("Hello world!\n");
    return 0;
}
```

编译 && 调试
```bash
$ gcc main.c -g && gdb ./a.out
```

![](/images/debugging-glibc-0.jpeg)
我们在printf处单步进入，已经可以跳转到相关的代码了
![](/images/debugging-glibc-1.jpeg)

## References
<https://www.gnu.org/software/hurd/faq/debugging_inside_glibc.html>
<https://www.gnu.org/software/libc/manual/html_node/Configuring-and-compiling.html>