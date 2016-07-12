[转自tyy2016 CSDN博客](http://blog.csdn.net/github_35384800/article/details/51815770)
# 探索CDN之一：初识CDN

本系列文章将从初学者视角，一步一步循序渐进地探索CDN的相关知识。本文为系列第一篇，主要是对CDN及相关内容的简单介绍和理解。主要内容如下：

[TOC]


-------------------

## 什么是CDN？
-------------------
CDN的英文全写是“Content Delivery Network”，中文译名为“内容分发网络”。下面是援引维基百科关于CDN的定义：
> A content delivery network or content distribution network is a globally distributed network of proxy servers deployed in multiple data centers. The goal of a CDN is to end-users with high availability and high performance.    —— <a href="https://en.wikipedia.org/wiki/Content_delivery_network" target="_blank"> [ 维基百科 ]

从这段“非正式”定义能够看出CDN的一些关键字：“proxy servers”，“HA and HP”等等。也就是说，CDN是依赖于一组代理服务器来提供高可用和高性能服务的网络架构。

CDN到底能干什么呢？举个例子即一目了然：

假设有一位身处陕西西安的用户想要访问某网站www.xxx.com提供的服务，这个网站的服务器部署在北京市，那么这个用户发送的请求需要跨过陕西省、山西省、河北省最终才能到达北京市。如此远的地理距离，会在一定程度上增加的请求响应时间，用户的感受可能会是网页加载速度变慢等。如下图：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn1.png)

为了缓解这个问题，该网站决定在陕西部署一组代理服务器，它们用来接收来自陕西的用户发送的请求，并将相应的内容返回。如下图：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn2.png)

这样做最直接的好处是缩短了地理空间，加速了用户请求的响应。当然陕西的服务器不会是重新搭建的，而是类似于镜像服务器，将位于北京的主服务器上的内容缓存下来，从而提供快速的访问。如下图：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn3.png)

一般情况下，当用户第一次访问该网站的某资源时，请求会被发送到位于西安的服务器上进行查找，如果查找失败，该服务器将去位于北京的主服务器上进行查找，并将返回的结果缓存下来，最终返回给用户。当用户再次访问该资源时，请求仍会被发送到位于西安的服务器上进行查找，并直接返回结果。

从图中可以看到，位于北京和位于西安的服务器的域名都是“www.xxx.com”，那么该如何区分？这就依赖于强大的DNS了。

-------------------
##  CDN和DNS？
-------------------
DNS的英文全写是“Domain Name System”，中文译名为“域名系统”。同样，下面援引维基百科的定义：
> The Domain Name System is a hierarchical decentralized naming system for computers, services, or any resource connected to the Internet or a private network.  —— <a href="https://en.wikipedia.org/wiki/Domain_Name_System" target="_blank"> [ 维基百科 ]

这段定义中最重要的就是“hierarchical decentralized”。这个特点将在后面进行讨论。

同样，DNS能够干什么呢？举个例子：

如果想要给某人打电话，必须翻阅电话本，按照他的名字找到相应的号码才能进行拨号。这样做的好处是解决了这么多类似、难记的电话号码的问题，人们只需要知道待呼的人的姓名即可。同样的，在网络世界，服务器间通信也要依赖于网络间的电话号码——IP地址，这些由数字构成的IP地址记忆起来非常困难，为了便于记忆，引入了域名的概念，它表示网络中一组服务器的名字，对应着一个IP地址。DNS就是为了完成这个任务而建立的系统。

在Unix/Linux系的系统，甚至在Windows系统中，可以找到/etc/hosts这样的文件，它记录了主机名和IP地址的简单映射，它的内容为：
```shell
	##
	# Host Database
	#
	# localhost is used to configure the loopback   interface when the system is booting. Do not change this entry.
	##
	127.0.0.1        localhost
	255.255.255.255  broadcasthost
	::1              localhost
```
当访问某网站时，首先会来该文件中查找相应的IP地址，如果有，则按照该地址进行访问，如果没有，则去请求DNS服务器以获得IP地址。

在上节的问题中，如何在网络中区分具有不同IP地址的同一个域名www.xxx.com，需要借助于DNS，使请求能够传递到正确的IP地址，即相应的服务器。如下图:
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn4.png)


那么如何尝试自己搭建简单的CDN呢？

-------------------
## 如何自己搭建简单的CDN（思路）？
-------------------
按照前述的架构，搭建简单的CDN需要以下材料：


项目     | 软件 | IP地址
-------- | --- | ---
http服务器 | Apache、 Nginx等 | 192.168.1.100
缓存服务器  | Squid | 192.168.1.101
用户客户端  | PC | 192.168.1.110

检查点Checkpoints：

	- 用户客户端访问http服务器得到正确返回
	- 修改/etc/hosts文件使用户客户端访问到缓存服务器并得到正确返回

在前面的测试成功后，可以将修改/etc/hosts的土办法改为使用DNS软件BIND来进行测试，Checkpoints同上。 
 
-------------------
