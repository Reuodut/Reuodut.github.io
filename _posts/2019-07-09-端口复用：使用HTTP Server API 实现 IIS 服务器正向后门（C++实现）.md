---
layout: post
title: "端口复用：使用HTTP Server API 实现 IIS 服务器正向后门（C++实现）"
date: 2019-07-09 
description: "端口复用"
tag: 工具开发
---   

### 0x01 前言

随着网络防火墙的发展，后门的生存环境变得越来越恶劣，不寻常端口间的通信受到防火墙的严格审查，在较高的安全环境下，服务器仅仅对外开放一两个端口，后门的连接通信变得难上加难，本文围绕如何构建一个适用于IIS 6及以上版本的端口复用后门进行叙述，程序使用C++及微软官方HTTP Server API 1.0进行开发，最终实现与IIS共享80/443端口，对特定的请求进行处理后返回响应的后门程序。端口复用基础原理知识这里将不会赘述，不熟悉的读者可以自行网上查阅相关介绍。

### 0x02 IIS 与 HTTP Server API的关系

在以前的Web应用中，一个Web应用只能绑定一个端口，若两个Web应用同时绑定一个端口，那么后者绑定端口时将会抛出端口被占用的异常信息。而现在却不同，微软在IIS 5.0以后的版本中提供了一种机制：Net.TCP Port Sharing(端口共享机制)，它允许多个应用同时使用一个端口。这种机制最终被封装在了HTTP.sys这个驱动中，微软对外开放了如何调用这种机制的API接口，也就是HTTP Server API，而非常让人高兴的是 IIS 也是基于此机制开发并运行的。如下图

![](/imag/20190709/2.png)

HTTP Server API 运行在用户模式中，也就是说任意用户都可以调用该API实现一个HttpListener，与 IIS 共享端口，但是前提是你必须拥有管理员权限。

上图整个过程描述如下：

（1）第一步，IIS 或者是自己写的程序（HttpListener）调用API ，向内核态的Http.sys注册一个URL前缀，这相当于向路由器添加一条路由规则，Http.sys就是这个路由器。MSDN示例：[点此](https://docs.microsoft.com/zh-cn/windows/win32/http/urlprefix-strings)

（2）第二步，Http.sys捕获到一个http请求，它将会根据自身的“路由表”找到该http请求的前缀所对应的应用，然后把请求分发给该应用。

（3）第三步，HttpListener接收到Http.sys转发来的请求，对其进行回应。

### 0x03 后门功能设计

微软官方有一个基于HTTP Server API 1.0版本的demo：[点此](https://docs.microsoft.com/zh-cn/windows/win32/http/http-server-sample-application)
本文基于该demo进行修改，设计一个功能较简单的后门，实现对http请求也就是80端口的共享，该api同时支持https协议，在此不作演示。

功能设计：
1. 与 IIS 共享80端口
2. 对传来的http请求进行身份验证，验证成功时进行回应
3. 命令执行
4. 流量加密

### 0x04 效果演示

环境：Windows server 2008 R2 + IIS7

![](/imag/20190709/2.gif)

### 0x05 后门实现及关键代码讲解

程序完善中，暂定7月底开源


