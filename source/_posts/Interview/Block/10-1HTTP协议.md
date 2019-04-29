---
title: HTTP&HTTPS协议
keywords: iOS面试
date: 2019-04-30 05:47:40
categories: 
  - 面试
tags:
  - 网络
comments: true
---

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-1.png)

# HTTP协议

`什么是HTTP？Http包括什么？`

A：HTTP是超文本传输协议

- 请求/响应报文:具体请求报文，响应报文的组成
- 连接建立流程
- HTTP的特点:无连接

`HTTP有哪些请求方式？`

- GET
- POST
- HEAD
- PUT
- DELETE
- OPTIONS

`GET和POST请求方式区别？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-4.png)

**安全性：**不应该引起Server端的任何状态的变化；满足安全性的方式：**GET、HEAD、OPTIONS**

**幂等性：**同一个请求方法**执行多次**和**执行**一次的效果完全效果；比如：PUT / DELETE/ GET

**可缓存性**：请求是否可以被缓存。比如：GET/HEAD

`你都了解哪些状态码？什么含义？`

- **1xx**：
- **2xx:** 200响应成功
- **3xx:** 301、302网络重定向
- **4xx:** 400、404客户端请求出问题
- **5xx:**500服务端问题

## 请求/响应报文

**请求报文格式：**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-2.png)

> 请求报文包括**<u>请求行</u>**：方法、ULR、协议版本；**<u>头部字段</u>**：以Key:Value组成；**<u>请求实体</u>**；
>
> 请求方法为Get方法的时候，是没有实体请求的。

**响应报文：**

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-3.png)

> 响应报文包括**<u>请求行</u>**：版本、状态码、状态码描述（短语）；**<u>头部字段</u>**：以Key:Value组成；**<u>响应实体</u>**；

## 连接建立流程

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-5.png)

> **TCP三次握手**：1.发送SYN同步报文；2.服务端收到连接请求SYN后，会返回一个SYN,ACK TCP报文；3.客户端再次回应一个ACK报文。
>
> **数据请求：**4.http发送请求报文 5.HTTP收到请求后，返回响应报文。完成响应后，进行四次挥手。
>
> **TCP四次挥手：**6.客户端主动发起连接断开，由客户端发起FIN终止报文给Server; 7.收到终止报文后，服务端会返回给客户一个确认报文ACK，此时由客户端到server端的链接就已经断开; 8.但是有可能server端还在给客户端传输数据，并没有断开，所以在某个时机有server向客户端发送终止报文FIN, AK报文，客户端收到后返回一个确认报文ACK。

**延伸问题：**

1. TCP连接建立的时候为什么是三次握手？
2. 四次挥手断开为什么会涉及两个方面的断开？

## HTTP的特点

- **无连接：**无连接表示HTTP连接有个建立连接和释放连接的过程。

  无连接解决：**HTTP的持久连接**

- **无状态：** 就是说多次发送HTTP请求的话，即使是同一个用户发送但是server是无法知晓的。

  解决办法：**Cookie/Session**

### HTTP持久连接

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-6.png)



- **非持久连接**：客户端和server交互的时候，每请求一次都需要重新创建连接，经历三次握手，四次挥手。

- **持久连接：** 客户端和server打开一条TCP通道，之后一定时间内，是在同一个TCP通道请求的。

`为什么HTTP提供了持久连接的方案？`

A：提升了网络请求的效率

**<u>持久连接涉及的头部字段：</u>**

- Connection: Keep-alive —> 表示客户端想要持久连接
- time ：20 —> 持久连接的事件是多少
- max : 10 —> 指这条TCP连接最多可以发生多少个HTTP请求和响应对。

`怎样判断一个请求是否结束？`

1. Content-length : 1024；从请求/响应报文的头部字段Content-length数据是否达到1024来判断
2. chunked: 对于POST请求在响应报文的头部字段中添加chunked,由于post请求数据是分段返回的，只有**最后一次的数据返回的响应头的chunked为空。**

`Charles抓包原理是怎样？`

本质就是：Http使用`中间人攻击`实现的。

### HTTP中间人攻击

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-7.png)

客户端和server建立的连接本来是直连的；中间人攻击，是有一个中间人接入客户端和server端。中间人可以篡改请求数据和返回数据。

# HTTPS

`HTTPS和HTTP有怎样的区别？`

HTTPS = HTTP + SSL/TLS：HTTP和SSL/TLS共同组成的，多了一个安全相关的模块。

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-8.png)

答：HTTPS是安全的HTTP，它的安全是由SSL、TLS提供的，SSL、TLS是插在应用层之下，传输层之上的。

`HTTPS的连接建立流程？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-10.png)

首先：

1.客户端向server发送一个报文，这个报文包括1).TLS版本号；2).支持的加密算法；3).随机数C

2.server返回客户端一个握手的报文，包括：1)商定的加密算法，随机数S，server证书。

3.客户端首先验证server证书，就是对server公钥验证。

4.组装**会话密钥**`：通过随机数C，S包括客户端产生的一个预主密钥进行会话密钥组装。

5.客户端再通过server的**公钥对预主密钥**进行加密传输。

6.后面在server端通过**私钥**解密得到预主密钥。

7.然后在server端通过随机数C，S和预主密钥进行会话密钥组装。

8.之后客户端发送给server端一个加密的握手消息。

9.server给客户端发送一个加密的握手消息。验证加密通道成功。

> 科普：
>
> **主密钥**是被客户机和服务器用于产生会话密钥的一个密钥。这个主密钥被用于产生客户端读密钥，客户端写密钥，服务器读密钥，服务器写密钥。主密钥能够被作为一个简单密钥块输出。
> **会话密钥**是指：当两个端系统希望通信，他们建立一条逻辑连接。在逻辑连接持续过程中，所以用户数据都使用一个一次性的会话密钥加密。在会话和连接结束时，会话密钥被销毁。

## 会话密钥

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-9.png)

会话密钥等于随机数S,C和预主密钥通过算法生成，一种对称加密。

### 加密有关

#### 非对称加密

`HTTPS使用了哪些加密算法？为什么？`

A:**非对称加密：**连接建立过程中使用非对称加密，非对称加密很耗时。

**对称加密：**后续通讯过程使用的对称加密。

`什么是非对称加密？`

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-11.png)

> 发送方对于发送的数据"Hello",发送方通过**公钥**对数据进行加密；经过**加密后的数据**传递给接收方；接收方拿到加密的数据经过**私钥**解密得到"Hello".
>
> 非对称加密中所使用的钥是不一样的，公钥加密就需要私钥解密，私钥加密就需要公钥解密。

#### 对称加密

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/10-1-12.png)



> 发送方对于发送的数据"Hello",发送方通过**私钥**对数据进行加密；经过**加密后的数据**传递给接收方；接收方拿到加密的数据经过**同样的私钥**解密得到"Hello".
>
> 对称加密中所使用的钥是一样的。



