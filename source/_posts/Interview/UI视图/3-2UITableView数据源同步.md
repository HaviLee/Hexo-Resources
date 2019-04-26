---
title: UITableView数据源同步
keywords: iOS面试
date: 2019-04-23 15:47:40
categories: 
  - 面试
tags:
  - UITableView数据源同步
comments: true
---

# 应用

新闻、资讯类的App中：

![场景](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/datasync.png)

删除操作属于用户的主动行为，应在主线程中。

# 数据源同步解决方案

 **<u>`如何解决TableView在多线程下修改、访问数据源的问题？`</u>**

- 并发访问、数据拷贝

![场景](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/conqueuedata.png)

{% label danger@弊端： %}涉及到数据拷贝，记录删除，会耗费大量的内存。

- 串行访问

![场景](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/serialdata.png)

{% label danger@弊端： %}要求子线程，特别耗时的时候，主线程删除会有延时。