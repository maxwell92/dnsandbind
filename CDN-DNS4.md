[转自tyy2016 CSDN博客](http://blog.csdn.net/github_35384800/article/details/51840594)
# 探索CDN之四：回到CDN实验

本系列文章将从初学者视角，一步一步循序渐进地探索CDN的相关知识。本文为系列第四篇，在前面探索的基础上，回到CDN实验，思考如何加入BIND完成实验，并思考如何集群化。主要内容如下：

[TOC]

-------------------

## 加入BIND？
-------------------

在上一次的试验中，使用了最土的办法——修改/etc/hosts来进行DNS解析，那么在上篇文章中对BIND有了初步了解后，将尝试采用BIND完成DNS解析。其中共修改了两次hosts文件，一次是修改用户客户端，使用户访问www.me.com的时候不是访问源服务器（123.206.5*.2*），而是访问作为缓存的Squid服务器（172.16.219.166）。另一次是修改了Squid容器，使它知道orig-www.me.com的IP地址。

由此可知，对于Squid缓存服务器来说，www.me.com可看做是orig-www.me.com的别名Alias（即源服务器），它们的解析地址可以在BIND里配置。另一方面，对于客户端来说，www.me.com可以看做是cdn.me.com的别名Alias（即Squid缓存服务器）。这样就产生了歧义——同样的www.me.com拥有两个不同IP的别名。扩展一下思路，位于不同地域的客户端访问同一个www.me.com时，同样会产生这样的歧义。这就是“智能DNS”要解决的问题——它会根据需求设定一些访问规则，例如对于不同的IP段给予不同的域名解析地址返回，这都可以通过BIND的视图view完成。个人理解，view就像namespace，不同namespace的同名区域可以有不一样的属性（例如IP地址）。BIND要求使用view时，所有的区域必须都放在view里。在修改完BIND之后，还需要将这两台服务器的DNS服务器名称改为BIND容器，这步将通过修改/etc/resolv.conf来完成。下面是实验的过程：

首先，创建view_172.17.0.5，它表示仅允许IP地址为172.17.0.5的客户端访问该view里的区域：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn25.png)
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn26.png)

并将默认的5个区域（root、0、127、255、localhost）都迁移至该view里。其中后面四个均可以通过WebMin来完成：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn27.png)

而root区域在WebMin里并没有提供相应的迁移，所以只能通过文件的方式修改：原本这5个区域的记录都位于/etc/bind/named.conf.default-zones里，而迁移到区域后它们就被”剪切“到了/etc/bind/named.conf.local里的view视图里。这里需要手动将root区域的内容”剪切“过去：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn28.png)

接着，创建view_172.17.0.6，同理，它表示仅允许IP地址为172.17.0.6的客户端访问该view里的区域。创建之后，需要创建me.com的区域，方法在上一篇文章里已经讲过，不再赘述，需要注意的地方有两点：1.要选择相应的view，2.在view_172.17.0.5里，www.me.com是cdn.me.com的别名，cdn.me.com的IP地址为172.16.219.170，而在view_172.17.0.6里，www.me.com是orig-www.me.com的别名，orig-www.me.com的IP地址是123.206.5*.2*：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn29.png)
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn30.png)

然后修改相应的hosts文件和resolv.conf文件：

```shell
	注释hosts文件里关于www.me.com的记录
	注释resolv.conf文件里原有的DNS服务器地址，添加新的DNS服务器地址：nameserver 172.17.0.4
```

最后在客户端进行测试：

在IP地址为172.17.0.5的机器上运行命令host www.me.com返回如下结果，它表示返回了源服务器地址：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn33.png)

在IP地址为172.17.0.6的机器上运行命令host www.me.com返回如下结果，它表示返回了Squid缓存服务器地址：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn32.png)

在IP地址既非172.17.0.5也非172.17.0.6的机器上运行命令host www.me.com返回如下结果，它表示BIND拒绝了请求：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn31.png)


-------------------

## 集群化的Squid？
-------------------

“集群化？为什么要集群化？难道一台服务器不够用么？”，“对，是不够用，并且。。。。额，它挂了怎么办？”

集群化带来的好处显而易见，集群化除了可以提供一个“可被看做高性能服务器整体”的特征外，还可以解决单点问题，提供高可用，并且具备良好的扩展性。这里不想讨论如何对源服务器进行集群化，那是一项非常耗时且复杂的工作（除了技术层面，还有业务的关系），思考一下如何完成Squid缓存服务器的集群化：假如部署三台Squid，它们同时工作，并且对外仅暴露一个端口。那么这样的架构可以为下图：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn34.png)

其中LB为Load Balancer，可以使用F5、LVS和轻量的Nginx实现。为了简便，本文中采用Nginx实现。

Nginx作为反向代理和负载均衡器需要修改其配置文件/etc/nginx.conf，主要添加如下字段：

```shell
	upstream backend {
             server 172.17.0.7;
             server 172.17.0.8;
             server 172.17.0.9;
         }
	server {

        listen       80;
        server_name  trffweb;

        location / {
             #反向代理的地址
             proxy_pass http://backend;     
        }
}

```

Nginx作为负载均衡器，它默认的访问策略是“轮询Round-Robin”，它也可以被设置为加权等策略。

而后面三个Squid服务器，它们的配置是完全一样的。在搭建成功后，使用客户端服务器进行测试：修改前面的cdn.me.com地址为Nginx服务器的地址，并在客户端服务器(172.17.0.6)执行两次host www.me.com命令：

第一台Squid服务器的日志(/var/log/Squid3/access.log)：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn35.png)

第二台Squid服务器的日志：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn351.png)

第三台Squid服务器的日志：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn36.png)


1. 扩展思考一下，这些Squid服务器之间缓存的内容是否要一致？如果要一致，该怎么做？
   如果这些Squid服务器缓存的内容不一致的话，带来的问题主要是这三台服务器并不是完全相同，假设其中一台宕机，其它的如果想要接替它的工作，由于缓存的内容不一致，对于并未在这台服务器上访问过的资源，它需要访问源主机请求并缓存返回，这只是诸多不便中的一条。更主要的是，缓存可以看做是该服务器的“状态”，即数据，具有状态的服务器在高可用、迁移等方便均存在不便，如何去状态化可能会是后续技术发展的趋势。

   就这里的场景而言，要想它们之间完成数据同步，有这样两种思路，一种还是各自保持各自的缓存，设定同步间隔，分出主从，每隔一段时间从服务器和主服务器完成同步。另一种是采用集中存储，无论是SAN，还是当前比较火的Ceph，都可以用来尝试解决这个问题。

2. 扩展一下，如果这三台Squid服务器一台为Parent，另两台为Sibling该如何配置？
   这个留着后面实验吧。 :)


-------------------

## 参考资料
-------------------

- [《DNS与BIND》](http://item.jd.com/11600174.html)
- [Deploying a DNS Server using Docker](https://www.damagehead.com/blog/2015/04/28/deploying-a-dns-server-using-docker/)
