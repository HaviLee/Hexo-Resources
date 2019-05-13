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

![4-5-1](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/https-link.png)

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

---------------------------------------

HTTP 请求报文由请求行、请求头部、空行 和 请求包体 4 个部分组成，如下图所示：

![HTTP 请求报文由请求行、请求头部、空行 和 请求包体 4 个部分组成](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/http-header.png)

　　下面对请求报文格式进行简单的分析：

　　请求行：请求行由方法字段、URL 字段 和HTTP 协议版本字段 3 个部分组成，他们之间使用空格隔开。常用的 HTTP 请求方法有 GET、POST、HEAD、PUT、DELETE、OPTIONS、TRACE、CONNECT;

　　● GET：当客户端要从服务器中读取某个资源时，使用GET 方法。GET 方法要求服务器将URL 定位的资源放在响应报文的[数据](http://ad.doubleclick.net/ddm/trackclk/N7442.5006CHINABYTE/B10313247.138166523;dc_trk_aid=310538354;dc_trk_cid=74205219)部分，回[送给](http://www.chinabyte.com/keyword/送给/)客户端，即向服务器请求某个资源。使用GET 方法时，请求参数和对应的值附加在 URL 后面，利用一个问号(“?”)代表URL 的结尾与请求参数的开始，传递参数长度受限制。例如，/index.jsp?id=100&op=bind。

　　● POST：当客户端给服务器提供信息较多时可以使用POST 方法，POST 方法向服务器提交数据，比如完成表单数据的提交，将数据提交给服务器处理。GET 一般用于获取/查询资源信息，POST 会附带用户数据，一般用于更新资源信息。POST 方法将请求参数[封装](http://www.chinabyte.com/keyword/封装/)在HTTP 请求数据中，以名称/值的形式出现，可以传输大量数据;

　　请求头部：请求头部由关键字/值对组成，每行一对，关键字和值用英文冒号“:”分隔。请求头部通知服务器有关于客户端请求的信息，典型的请求头有：

　　● User-Agent：产生请求的[浏览器](http://www.chinabyte.com/keyword/浏览器/)类型;

　　● Accept：客户端可识别的响应内容类型列表;星号 “ * ” 用于按范围将类型分组，用 “ */* ” 指示可接受全部类型，用“ type/* ”指示可接受 type 类型的所有子类型;

　　● Accept-Language：客户端可接受的自然语言;

　　● Accept-Encoding：客户端可接受的编码压缩格式;

　　● Accept-Charset：可接受的应答的字符集;

　　● Host：请求的主机名，允许多个[域名](http://www.chinabyte.com/keyword/域名/)同处一个IP 地址，即虚拟主机;

　　● connection：连接方式(close 或 keepalive);

　　● Cookie：[存储](http://storage.chinabyte.com/)于客户端扩展字段，向同一域名的服务端发送属于该域的cookie;

　　空行：最后一个请求头之后是一个空行，发送回车符和换行符，通知服务器以下不再有请求头;

　　请求包体：请求包体不在 GET 方法中使用，而是在POST 方法中使用。POST 方法适用于需要客户填写表单的场合。与请求包体相关的最常使用的是包体类型 Content-Type 和包体长度 Content-Length;

　　HTTP 响应报文

　　HTTP 响应报文由状态行、响应头部、空行 和 响应包体 4 个部分组成，如下图所示：

![img](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/http-response.png)

　　下面对响应报文格式进行简单的分析：

　　状态行：状态行由 HTTP 协议版本字段、状态码和状态码的描述文本 3 个部分组成，他们之间使用空格隔开;

　　● 状态码由三位数字组成，第一位数字表示响应的类型，常用的状态码有五大类如下所示：

　　1xx：表示服务器已接收了客户端请求，客户端可继续发送请求;

　　2xx：表示服务器已成功接收到请求并进行处理;

　　3xx：表示服务器要求客户端重定向;

　　4xx：表示客户端的请求有非法内容;

　　5xx：表示服务器未能正常处理客户端的请求而出现意外错误;

　　● 状态码描述文本有如下取值：

　　200 OK：表示客户端请求成功;

　　400 Bad Request：表示客户端请求有语法错误，不能被服务器所理解;

　　401 Unauthonzed：表示请求未经授权，该状态代码必须与 WWW-Authenticate 报头域一起使用;

　　403 Forbidden：表示服务器收到请求，但是拒绝提供服务，通常会在响应正文中给出不提供服务的原因;

　　404 Not Found：请求的资源不存在，例如，输入了错误的URL;

　　500 Internal Server Error：表示服务器发生不可预期的错误，导致无法完成客户端的请求;

　　503 Service Unavailable：表示服务器当前不能够处理客户端的请求，在一段时间之后，服务器可能会恢复正常;

　　响应头部：响应头可能包括：

　　Location：Location响应报头域用于重定向接受者到一个新的位置。例如：客户端所请求的页面已不存在原先的位置，为了让客户端重定向到这个页面新的位置，服务器端可以发回Location响应报头后使用重定向语句，让客户端去访问新的域名所对应的服务器上的资源;

　　Server：Server 响应报头域包含了服务器用来处理请求的软件信息及其版本。它和 User-Agent 请求报头域是相对应的，前者发送服务器端软件的信息，后者发送客户端软件(浏览器)和[操作系统](http://soft.chinabyte.com/os/)的信息。

　　Vary：指示不可缓存的请求头列表;

　　Connection：连接方式;

　　对于请求来说：close(告诉[ WEB 服务器](http://www.chinabyte.com/keyword/web服务器/)或者[代理服务器](http://www.chinabyte.com/keyword/代理服务器/)，在完成本次请求的响应后，断开连接，不等待本次连接的后续请求了)。keepalive(告诉WEB服务器或者代理服务器，在完成本次请求的响应后，保持连接，等待本次连接的后续请求);

　　对于响应来说：close(连接已经关闭); keepalive(连接保持着，在等待本次连接的后续请求); Keep-Alive：如果浏览器请求保持连接，则该头部表明希望WEB 服务器保持连接多长时间(秒);例如：Keep-Alive：300;

　　WWW-Authenticate：WWW-Authenticate响应报头域必须被包含在401 (未授权的)响应消息中，这个报头域和前面讲到的Authorization 请求报头域是相关的，当客户端收到 401 响应消息，就要决定是否请求服务器对其进行验证。如果要求服务器对其进行验证，就可以发送一个包含了Authorization 报头域的请求;

　　空行：最后一个响应头部之后是一个空行，发送回车符和换行符，通知服务器以下不再有响应头部。

　　响应包体：服务器返回给客户端的文本信息;

　　HTTP 工作原理

　　HTTP 协议采用请求/响应模型。客户端向服务器发送一个请求报文，服务器以一个状态作为响应。

　　以下是 HTTP 请求/响应的步骤：

　　● 客户端连接到web服务器：HTTP 客户端与web服务器建立一个 TCP 连接;

　　● 客户端向服务器发起 HTTP 请求：通过已建立的TCP 连接，客户端向服务器发送一个请求报文;

　　● 服务器接收 HTTP 请求并返回 HTTP 响应：服务器解析请求，定位请求资源，服务器将资源副本写到 TCP 连接，由客户端读取;

　　● 释放 TCP 连接：若connection 模式为close，则服务器主动关闭TCP 连接，客户端被动关闭连接，释放TCP 连接;若connection 模式为keepalive，则该连接会保持一段时间，在该时间内可以继续接收请求;

　　● 客户端浏览器解析HTML内容：客户端将服务器响应的 html 文本解析并显示;

　　例如：在浏览器地址栏键入URL，按下回车之后会经历以下流程：

　　1、浏览器向[ DNS 服务器](http://www.chinabyte.com/keyword/DNS服务器/)请求解析该 URL 中的域名所对应的 IP 地址;

　　2、解析出 IP 地址后，根据该 IP 地址和默认端口 80，和服务器建立 TCP 连接;

　　3、浏览器发出读取文件(URL 中域名后面部分对应的文件)的HTTP 请求，该请求报文作为 TCP 三次握手的第三个报文的数据发送给服务器;

　　4、服务器对浏览器请求作出响应，并把对应的 html 文本发送给浏览器;

　　5、释放 TCP 连接;

　　6、浏览器将该 html 文本并显示内容;

　　HTTP 无状态性

　　HTTP 协议是无状态的(stateless)。也就是说，同一个客户端第二次访问同一个服务器上的页面时，服务器无法知道这个客户端曾经访问过，服务器也无法分辨不同的客户端。HTTP 的无状态特性简化了服务器的设计，使服务器更容易支持大量并发的HTTP 请求。

　　HTTP 持久连接

　　HTTP1.0 使用的是非持久连接，主要缺点是客户端必须为每一个待请求的对象建立并维护一个新的连接，即每请求一个文档就要有两倍RTT 的开销。因为同一个页面可能存在多个对象，所以非持久连接可能使一个页面的下载变得十分缓慢，而且这种短连接增加了[网络](http://ad.doubleclick.net/ddm/trackclk/N7442.5006CHINABYTE/B10313247.138166524;dc_trk_aid=310538621;dc_trk_cid=74205219)传输的负担。HTTP1.1 使用持久连接keepalive，所谓持久连接，就是服务器在发送响应后仍然在一段时间内保持这条连接，允许在同一个连接中存在多次数据请求和响应，即在持久连接情况下，服务器在发送完响应后并不关闭TCP 连接，而客户端可以通过这个连接继续请求其他对象。

　　HTTP/1.1 协议的持久连接有两种方式：

　　● 非流水线方式：客户在收到前一个响应后才能发出下一个请求;

　　● 流水线方式：客户在收到 HTTP 的响应报文之前就能接着发送新的请求报文;

[长连接和短连接]: https://www.cnblogs.com/gotodsp/p/6366163.html

