---
layout: post
title: "Linux hook：Ring3下动态链接库.so函数劫持"
date: 2020-09-09 
description: "Ring3下动态链接库.so函数劫持"
tag: Linux攻防
---  

### 一、动态链接库函数劫持原理
 Unix操作系统中，程序运行时会按照一定的规则顺序去查找依赖的动态链接库，当查找到指定的so文件时，动态链接器(/lib/ld-linux.so.X)会将程序所依赖的共享对象进行装载和初始化，而为什么可以使用so文件进行函数的劫持呢？

这与LINUX的特性有关，先加载的so中的全局符号会屏蔽掉后载入的符号，也就是说如果程序先后加载了两个so文件，两个so文件定义了同名函数，程序中调用该函数时，会调用先加载的so中的函数，后加载的将会屏蔽掉；所以要实现劫持，必须要抢得先机，

环境变量LD_PRELOAD以及配置文件/etc/ld.so.preload就可以让我们取得这种先机，它们可以影响程序的运行时的链接（Runtime linker），它允许你定义在程序运行前优先加载的动态链接库,我们只要在通过LD_PRELOAD加载的.so中编写我们需要hook的同名函数，即可实现劫持！从下图中我们使用strace可以看到，优先加载了LD_PRELOAD指明的.so

![](/imag/20200909/1.png)
    


#### 劫持普通函数当然没有什么意思了！我们要劫持的是系统函数！

我们知道，Unix操作系统中对于GCC而言，默认情况下，所编译的程序中对标准C函数（fopen、printf、execv家族等等函数）的链接，都是通过动态链接方式来链接libc.so.6这个函数库的，我们只要在加载libc.so.6之前加载我们自己的so文件就可以劫持这些函数了。

### 二、Demo
我们从一个简单的c程序（sample.c）开始

下面的代码标准调用fopen函数，并检查返回值

    #include <stdio.h>
    int main(void) {
        printf("Calling the fopen() function...\n");
        FILE *fd = fopen("test.txt","r");
        if (!fd) {
            printf("fopen() returned NULL\n");
            return 1;
        }
        printf("fopen() succeeded\n");
        return 0;
    }

编译执行

    $ gcc -o sample sample.c

    $ ./sample 
    Calling the fopen() function...
    fopen() returned NULL

    $ touch test.txt

    $ ./sample 
    Calling the fopen() function...
    fopen() succeeded

开始编写我们自己的so动态库

    #include <stdio.h>
    FILE *fopen(const char *path, const char *mode) {
        printf("This is my fopen!\n");
        return NULL;
    }

编译成.so

    gcc -Wall -fPIC -shared -o myfopen.so myfopen.c

设置环境变量后执行sample程序，我们可以看到成功劫持了fopen函数，并返回了NULL

    $ LD_PRELOAD=./myfopen.so ./sample
    Calling the fopen() function...
    This is my fopen!
    fopen() returned NULL

当然 ，使fopen始终返回null是不明智的，我们应该在假的fopen函数中还原真正fopen的行为，看下面代码

这回轮到 dlfcn.h 出场，来对动态库进行显式调用，使用dlsym函数从c标准库中调用原始的fopen函数，并保存原始函数的地址以便最后返回 恢复现场

    #define _GNU_SOURCE
    #include <stdio.h>
    #include <dlfcn.h>
    FILE *fopen(const char *path, const char *mode) {
        printf("In our own fopen, opening %s\n", path);
        FILE *(*original_fopen)(const char*, const char*);
        original_fopen = dlsym(RTLD_NEXT, "fopen");
        return (*original_fopen)(path, mode);
    }

Tips：
如果dlsym或dlvsym函数的第一个参数的值被设置为RTLD_NEXT，那么该函数返回下一个共享对象中名为NAME的符号（函数）的运行时地址。
下一个共享对象是哪个，依赖于共享库被加载的顺序。dlsym查找共享库顺序如下：
①环境变量LD_LIBRARY_PATH列出的用分号间隔的所有目录。
②文件/etc/ld.so.cache中找到的库的列表，由ldconfig命令刷新。
③目录usr/lib。
④目录/lib。
⑤当前目录。

 编译：

    gcc -Wall -fPIC -shared -o myfopen.so myfopen.c -ldl

执行：调用原始函数，劫持成功！

    $ LD_PRELOAD=./myfopen.so ./sample
    Calling the fopen() function...
    In our own fopen, opening test.txt
    fopen() succeeded

### 三、需要注意的问题以及LD_PRELOAD hook的应用

需要注意的问题

1.so文件加载及函数劫持的顺序。在很多情况下，在你进行劫持之前，系统中已经有其他组件也对该函数进行了劫持，那么就要特别注意so加载的顺序，一定要在其他组件的so库加载前加载自己的so库，否则你的hook函数将会被忽略。

2.保持原本函数的完备性与业务的兼容性。被hook的函数一定要hook结束时进行返回，返回前自己的执行逻辑中不能过度延时，过度延时可能造成原有的业务逻辑失败。使用RTLD_NEXT 句柄，维持原有的共享库调用链。

#### 应用一：HIDS入侵检测系统
劫持libc库

优点: 性能较好, 比较稳定, 相对于LKM更加简单, 适配性也很高, 通常对抗web层面的入侵.
缺点: 对于静态编译的程序束手无策, 存在一定被绕过的风险.

#### 应用二：rootkit恶意软件
已经有多种恶意软件应用了此技术，常见的有cub3、vlany、bdvl等
之后的几篇文章我将会通过分析以上几款恶意软件来揭秘.so共享库劫持技术的具体应用，敬请期待！



### 转载请注明出处，本文首发于 https://www.freebuf.com/articles/system/247462.html

安全编程交流：NzgyNDIxODg3


