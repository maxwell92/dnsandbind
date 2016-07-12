[转自tyy2016 CSDN博客](http://blog.csdn.net/github_35384800/article/details/51832883)
# 探索CDN之三：DNS与BIND

本系列文章将从初学者视角，一步一步循序渐进地探索CDN的相关知识。本文为系列第三篇，简单介绍DNS的知识、BIND的概念及简单的BIND实验。主要内容如下：

[TOC]

-------------------

## 进一步聊聊DNS？
-------------------

在上一篇文章里，曾将DNS看做是一个“电话本”。通过手动修改/etc/hosts的方式来改变域名和IP地址间的映射非常“简单粗暴”。随着互联网的发展，全世界每天都有成千上万个网站域名出现和消失，全靠手动是不现实的。于是出现了更为“智能”和“自动化”的DNS体系。

最初HOSTS.txt文件由SRI的网络信息中心（NIC）进行更新和分发，但是存在HOSTS.txt的限制（同一文件里名称冲突、分布式一致性），和分发带来的流量和负载等问题。现行的DNS规范为RFC1034和RFC1035。

DNS数据库的结构类似于Unix/Linux系统的文件系统结构——倒置的树状结构。树中每个节点都拥有一个最长为63字节的标签，域名是从该域的root节点开始，一直回溯到整棵树的root节点的标签序列，并以.号分隔。一个绝对域名，类似于Unix系统中的绝对路径，用来表示该节点在层次结构中的位置，通常也被称为完全限定域名（FQDN，Fully Qualified Domain Name)。如下图：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn11.png)

这棵树的每个子树都代表了整个数据库的一部分，称作“域domain”，也即“域命名空间”。每个域目录又可以被进一步划分为额外的部分，称作“子域subdomain”。DNS的每个域都可以被分解成若干个子域，并均可被授权给不同的组织分散管理。域和子域都有相应的管理者，它们不必一样。例如edu域由EDUCAUSE管理，而它的子域xupt.edu会委托XUPT进行管理。要将子域授权给某个机构进行管理，需要创建一个“区域zone”，它是域命名空间中一段可以自治的区域，xupt.edu区域包含了所有以xupt.edu结尾的域名，而edu区域包含所有以edu结尾的域名，但不包含xupt.edu结尾的域名。xupt.edu这个区域可以进一步划分为lib.xupt.edu等子域，如果将其委托出去又可以构成新的独立区域。

原始的顶级域将Internet域分为了7个域，com、edu、cn等。

域中包含的主机和子域的域名，位于该域所属的命名空间的子树中，叶子节点往往表示的是某台主机。网络上的每台主机都有一个域名，它指向该主机的相关信息。同时还可能拥有多个别名（Alias)。

DNS域名系统是一个分布式数据库，允许对各个部分采用客户端/服务器模式进行本地控制，辅以复制和缓存等机制，使它拥有了健壮性和充足的性能。DNS的客户端和服务器分别叫做“解析器resolver"和”名称服务器nameserver“。

名称服务器nameserver通常只拥有域命名空间的一部分的完整信息，即区域。它的内容是从文件或另外的名称服务器加载而来。加载过后，这个名称服务器宣称对该区域具有“权威authority”。这样做的好处是：一个域的信息可能超出了一个名称服务器所需要的量，区域通过授权与原域划分了界限。

DNS定义了两种类型的名称服务器：primary master和secondary master。primary master从主机上读取数据，而secondary master从该区域的权威名称服务器(master，通常是该区域的primary master)上读取区域数据，也可以从另一个secondary上去读。获取区域数据的过程称作“区域传输zone transfer”，secondary master目前更多地被称为slave服务器。master服务器和slave服务器都是该区域的权威，但并不唯一，意思是区域A的master服务器可能是区域B的slave服务器。master服务器从本地加载的区域数据文件被称为“区域数据文件zone datafile”，slave常被配置为从master同步该文件，并更新数据。

解析器resolver处理的任务有：查询名称服务器、解释响应信息、将信息返回给查询它的程序。

而名称服务器在域命名空间中查找不以自己为权威的区域的数据的过程被称为名称解析（name resolution）。解析的方法有两种：递归和迭代。它们的区别如下表：

解析方法     | 特点 
-------- | --- 
递归 | 大部分解析负担在一个名称服务器上，由它负责查询结果，它发出的查询跟它收到来自解析器的完全一样
迭代  | 在本地数据中找出与所查询的名称服务器最接近的名称服务器的名称和地址，并返回给查询者以供继续查询，工作量较小

递归方法图示：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn9.png)

迭代方法图示：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn10.png)

在Internet域命名空间上有一部分以地址为标签的域命名空间，即in-addr.arpa域。在读域名时，IP地址应该倒过来读，假设lib.xupt.edu的IP地址为202.114.65.156，那么它在in-addr.arpa域对应的节点为156.65.114.202。


-------------------

## BIND及其简单实验？
-------------------

BIND的英文全写是“Berkeley Internet Name Domain”，它是目前普及最广的DNS实现。要想使用BIND可以获取其二进制包、源码包，也可以通过Docker容器来完成。这里使用Docker化的BIND 9进行实验。

首先，在本地计算机上运行以下命令：

```shell
	host www.baidu.com
```

可以看到返回结果为下图，它表示默认的DNS服务器对www.baidu.com进行了解析，发现它是www.a.shifen.com的别名，并得到了它的IP地址。

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn12.png)

接着，运行BIND容器，运行以下命令：

```shell
	host www.baidu.com 172.17.0.4
```

返回结果如下图，这使用了自己搭建的BIND作为DNS服务器进行的解析。发现这次的IP跟上次的不同？输入浏览器发现它仍然是百度主页。

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn13.png)

该BIND的Docker镜像里已经预置了webmin（一款图形化的Linux管理工具，非常便于远程管理计算机），通过它可以尝试配置BIND。图形化的配置过程比较清晰，基本过程是：

```sequence
	创建一个master zone->添加Address->添加Name Alias
```

首先，通过浏览器访问webmin，登录并选择BIND DNS Server：

然后，在下方的区域信息处点击创建主区域Creating master zone，并选择正向Forward类型，配置Domain Name/Network为example.com，Master server为：ns.example.com，最后应用Apply，如下图：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn15.png)
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn16.png)
接着，在前面的zone列表里选择example.com，并创建两个Address，一个Name为webserver.example.com，Address为192.168.1.1。另一个为ns.example.com，Address为172.17.0.4，修改好后Apply Configuration，如下图：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn17.png)
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn18.png)

最后，同样选择example.com，并创建一个Name Alias，Name为www.example.com，Real Name为：webserver.example.com，如下图：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn19.png)

执行下面的命令：

```shell
	host www.example.com 172.17.0.4
```

结果为：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn20.png)








-------------------

## 参考资料
-------------------

- [《DNS与BIND》](http://item.jd.com/11600174.html)
- [Deploying a DNS Server using Docker](https://www.damagehead.com/blog/2015/04/28/deploying-a-dns-server-using-docker/)
