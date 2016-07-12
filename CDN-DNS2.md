[转自tyy2016 CSDN博客](http://blog.csdn.net/github_35384800/article/details/51818450)
# 探索CDN之二：Squid与简单CDN实验

本系列文章将从初学者视角，一步一步循序渐进地探索CDN的相关知识。本文为系列第二篇，进一步介绍CDN的知识、Squid缓存服务器和简单的CDN实验。主要内容如下：

[TOC]

-------------------

## 进一步聊聊CDN？
-------------------

在上一篇文章里，曾讲到CDN是“内容分发网络”，依赖于一组代理服务器来提供高可用、高性能的服务。而建立在TCP/IP基础上的互联网，它的设计理念之一是“网络的中立和无控制”，以便于更好、更快地将数据包进行传输。互联网不会对传输内容进行任何优化。

网络层和应用层之间的“磨合”存在一些可能造成“拥堵”的地方：

1. “第一公里”。它指的是某网站服务器接入互联网后的带宽，直接影响了用户的访问速度和并发访问量。当用户请求量大于带宽时，就会造成拥堵。

2. “最后一公里”。它指的是用户接入互联网的带宽。类似于“第一公里”，假设服务器返回的数据量大于带宽，用户同样会感觉网络拥堵。目前由于光纤等技术的普及，这一问题已得到很大程度上的解决了。

3. 对等互联关口。它指的是不同运营商（ISP）间的互联互通点。一般两个运营商间仅有2~3个互联互通点，可想而知这些点上负载了巨大的流量，容易造成拥堵。

4. 长途骨干传输。用户发送请求到远方的网站服务器，会经过用户所在接入网、用户所在城域网、骨干网直到网站所在IDC等。存在长距离传输时延和骨干网拥塞问题。

CDN的出现对于缓解上述问题具有重大意义。在上篇文章里，曾举了www.xxx.com的例子来简介CDN的工作原理，具体来说，在CDN出现以前用户访问网站服务器的过程为：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn5.png)

CDN网络在用户和服务器间添加了Cache层，将用户的访问请求引导到最接近用户的网络“边缘”Cache站点而不是服务器源站点，通过接管DNS来实现，下图是使用CDN缓存网站内容后的访问过程：

![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn6.png)

使用了CDN之后，用户端和被加速的服务器端无需做任何改变，仅需要修改访问过程中的域名解析部分即可。

CDN代表了一种基于质量和秩序的网络服务模式，它包括分布式存储、负载均衡、网络请求重定向和内容管理4个要件，其中网络请求重定向和内容管理是CDN的核心。位于网络边缘的缓存服务器，它与用户仅有“一跳”之隔。

实现CDN的技术手段有：高速缓存、镜像服务器。可工作于DNS解析（准确率大于95%）和HTTP请求重定向（准确率大于99%）两种方式。一般情况下，各Cache服务器流入用户流量与Cache服务器向源服务器取数据量之比在2:1到3:1之间，即分担50%到70%的到原始网站重复访问数据数据量（主要是图片、流媒体文件等）；而对于镜像服务器，除数据同步的流量，其余均在本地完成。

CDN的优点是：

- 多域名加载资源。一般情况下浏览器对单个域名下的并发请求数（文件加载）进行限制，通常最多有4个，第5个只有前面的加载完毕了才能加载，而CDN文件是存放在不同IP的，对于浏览器可同时加载多个所需的所有文件，从而提高页面加载速度。
- 缓存加速。
- 高效。CDN提供更高的效率、更低的网络延时和更小的丢包率。
- 分布式数据中心。
- 版本控制。可以通过特定版本号从CDN加载响应的文件。
- 数据统计。一般CDN提供商都会提供访问统计等数据统计服务。
- 防止攻击。可有效防止DDoS攻击。

-------------------

## Squid缓存服务器？
-------------------

采用高速缓存的CDN需要缓存服务器，常用的缓存服务器有Squid等。Squid是一个Web缓存代理，它支持HTTP、HTTPS、FTP和其他。它可以通过缓存和重用频繁请求的web页面来达到减少带宽和提高响应时间的效果。使用Squid对于IPs、网站和内容分发提供者非常有益，其中Squid使得内容发布者和流媒体开发者在世界各地发布内容变得更加容易。CDN提供商可以购买廉价的运行Squid的PC硬件，并依照特定策略围绕internet部署来提供更加节约和高效的大数据量服务。

Squid的安装可以通过包管理系统、源码、二进制包等方式来安装。在Docker热潮中，同样出现了Docker化得Squid镜像。

Squid功能强大，它的配置也是较为复杂的。以Squid 3.3为例，它的配置说明可以参照[squid configure](http://www.squid-cache.org/Versions/v3/3.3/cfgman/)的内容说明。这里将以一个较为简单的配置文件简单进行介绍，更多配置项将在后续探索。


```shell
	http_port 80 accel ignore-cc defaultsite=www.me.com vhost # 设定监听端口80（默认是3128），并指定工作方式
	acl all src all # 访问控制，这里允许全部
	cache_peer orig-www.me.com parent 9999 0 no-query originserver name=wwwip # 设定源服务器域名及端口号
	cache_peer_access all allow all # 源服务器访问控制
	http_access allow all # http访问控制
	visible_hostname cdn.me.com # 对外主机名
	cache_mem 1 GB # 缓存大小

```

上面的配置是最简单的配置。具体来说其中：

- http_port，表示Squid用来监听HTTP客户端请求的socket地址。默认是3128，可以自定义。accel 属于工作模式，它表示Squid工作在加速器(accelerator) / 反向代理模式下。ignore-cc表示忽略请求缓存控制头。它是accelerator模式下的选项。defaultsite表示默认主机名，即当头部没有展现主机名时需要加速的默认主机。vhost表示开启使用HTTP 1.1 头部支持虚拟域名支持。

- acl，用来定义访问列表。可以灵活设定多种规则进行访问控制。

- cache_peer 它表示指定缓存对端，格式为：cache_peer hostname type http-port icp-port [options]。其中type可以为parent、sibling或者multicast。后面的http-port表示对端接受HTTP请求的端口，icp-port表示查询邻居缓存的端口，为0表示不支持ICP和HTCP，所以在后面还会加上no-query表示不支持邻近ICP查询。originserver表示该服务器为源服务器。

-------------------

## 简单的CDN实验？
-------------------

在上一篇文章的最后一部分提出了简单的CDN实验。实验所需环境如下：

项目     | 软件 | IP地址
-------- | --- | ---
http服务器 | Nginx | 123.206.5*.2*
缓存服务器  | Squid | 192.168.1.106
用户客户端  | PC | 192.168.1.104

首先，http服务器上启动Docker容器Nginx，对外开放9999端口，并承载内容为“Hello Origin Server”的index.html文件。

经测试，通过缓存服务器和用户客户端均可以curl访问到结果“Hello Origin Server”。

然后在缓存服务器上启动Docker容器Squid，配置文件如上节示例，并在容器的/etc/hosts文件中添加"123.206.5*.2* orig-www.me.com"。

接着在用户客户端上修改/etc/hosts文件，添加记录"192.168.1.106 www.me.com"，以进行简单的DNS。

最后在用户客户端上访问www.me.com:3128可以得到结果"Hello Origin Server"。

若使用wget保存www.baidu.com主页为index.html到http服务器，并使用Nginx启动，按照上述的方法，最后一步使用curl -I查看返回的头，第一次缓存没有命中，返回如下：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn7.png)

第二次缓存命中，返回如下：
![](http://7xiwbf.com1.z0.glb.clouddn.com/cdn8.png)


-------------------

## 参考资料
-------------------

- [《CDN技术详解》](http://item.jd.com/11600174.html)
- [Tim Berners Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee)
- [CDN技术原理](http://blog.csdn.net/liuhongxiangm/article/details/8785312)
- [Squid](http://www.squid-cache.org/)
