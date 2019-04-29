---
title: DNS & Session/Cookie
keywords: iOS面试
date: 2019-04-30 07:47:40
categories: 
  - 面试
tags:
  - 网络
comments: true
---

# DNS

`是否了解DNS解析，其过程是什么？`

A:域名到IP地址的映射，DNS解析**请求采用UDP数据报，且名文**。

下面是解析过程

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-1.png)

> 客户端会通过DNS协议去向DNS服务器请求对相应域名的解析，DNS会把对应的IP返回给客户端，再由客户端向IPserver发起请求。

`DNS查询方式有哪些？`

- 递归查询
- 迭代查询

**递归查询**：我去给你问一下

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-2.png)



> 客户端向DNS域名服务器发起请求的时候，会首先查询本地DNS服务器，如果知道就返回，不知道的话，会去询问根域DNS；一步步的想上查询。

**迭代查询：**我告诉你谁可能知道

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-3.png)

> 客户端向DNS域名服务器发起请求的时候，会首先查询本地DNS服务器，本地DNS说我不知道，根域DNS可能知道，然后本地去查根域DNS，然后根域说我也不知道，你可以去查顶级DNS。

`DNS解析存在哪些常见的问题？`

- DNS劫持问题
- DNS解析转发问题

**DNS劫持**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-4.png)

> 客户端向本地DNS服务器查询某个域名对应的IP地址，由于DNS解析是通过TCP且明文查询的，有可能被窃听，如果有个钓鱼DNS，就可能被放回一个错误的IP地址，访问一个错误的网站。

`DNS劫持和HTTP的关系是怎样的？`

没有关系，DNS解析是发生HTTP建立之前，DNS解析请求使用UDP数据报，端口号为53；

**DNS解析转发**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-5.png)

> 比如客户端询问本地服务器，比如使用手机访问慕课网，www.imooc.com，某些服务商为了节省资源，会把域名转发给其它DNS服务器，另外的DNS服务器会去权威DNS请求解析；权威DNS服务器为不同的网络设置了固定的网络比如；移动：2.2.2.2，电信是3.3.3.3；因此最终返回了3.3.3.3。但是我们手机网络是移动的，造成跨网访问，因此会有缓慢情况。

`怎么解决DNS劫持？`

- httpDNS
- 长连接

**httpDNS:**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-6.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-7.png)

> 比如客户端向httpDNS服务进行GET请求，我们是通过IP直连的方式向httpDNS请求的，httpDNS会得到需要的IP地址，就可以进行后面的访问。

**长连接**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-8.png)

> 有客户端和客户端API server,在他们中间有个长连server;客户端和长链server之间通过长链通道沟通，而在长链Server和API Server之间通过内网专线通信。规避了公网DNS解析劫持。

# Session & Cookie

session和cookie都是对http无状态特点的补偿。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-9.png)

> 我们客户端对server进行网络请求的时候，server是无法记住用户的。再次请求的话，由于http无状态特点；

## Cookie

Cookie是用来记住用户状态，区分用户；**状态保存在客户端的**。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-11.png)

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-10.png)

运作流程：

> 1.客户端发送一个http请求对应的server，server会生成一个cookie，然后通过http的响应报文，把cookie携带上返回给客户端，客户端保存下来这个cookie。
>
> 2.server生成的cookie是如何在响应报文中的，在响应报文的首部字段中，通过添加setCookie传递给客户端。
>
> 3.后续的http请求中，将保存下来的cookie添加到http请求的头部字段中，传给server进行处理。

#### 如何修改Cookie?

- 新的cookie可以覆盖旧cookie
- 覆盖规则：name/path/domain等必须和原cookie保持一致才可以

#### 如何删除Cookie?

- 新的cookie可以覆盖旧cookie
- 覆盖规则：name/path/domain等必须和原cookie保持一致才可以
- 设置cookie的**expires=过去的一个时间点**，或者**maxAge=0**;让它失效

#### 怎样保证cookie的安全？

- 对cookie加密处理
- 只在https上携带Cookie
- 设置Cookie为httpOnly,防止跨站脚本攻击

## Session

Cookie是用来记住用户状态，区分用户；**状态保存在服务器端的**。

<u>Session和Cookie有怎样的联系？</u>

A：Session需要依赖Cookie机制

**Session工作流程：**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-12.png)

> 1.客户端和服务端进行数据传输，客户端发送一个http请求报文：服务端1）记录用户状态，2）生成一个session id.
>
> 2.然后将session id单发给客户端，采用的方式是通过 **Set-Cookie: session id**方式；
>
> 3.客户端在后续请求中，在请求的头部字段中设置session id.设置Cookie：seesion id;

# 总结：

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-3-13.png)