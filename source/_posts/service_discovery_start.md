---
abbrlink: service-discovery-start
title: 服务发现揭秘
date:  2021-11-05
description: 随着互联网业务的发展，互联网用户基数越来越大，对于服务提供者来说带来的并发压力也越来越大。以往通过单机提供服务的方式已经被淘汰，多机冗余、负载均衡等名词也已经成为当前服务开发的基本要求。如何感知到一个服务的多个节点，如何平滑的对于服务进行扩容，如何保证负载均衡的高效性，传统的负载均衡基础设施面对越来越多的挑战。而服务发现技术的出现，对于上述问题带来了一个更有的解决方案。
typora-copy-images-to: ../images
typora-root-url: ..
categories:
- Micro Service
---

## 1. 问题描述

随着互联网业务的发展，互联网用户基数越来越大，对于服务提供者来说带来的并发压力也越来越大。以往通过单机提供服务的方式已经被淘汰，多机冗余、负载均衡等名词也已经成为当前服务开发的基本要求。如何感知到一个服务的多个节点，如何平滑的对于服务进行扩容，如何保证负载均衡的高效性，传统的负载均衡基础设施面对越来越多的挑战。而服务发现技术的出现，对于上述问题带来了一个更有的解决方案。

![](/images/balancer_classic.png)

**图 1.1**

传统的负载均衡服务就如 **图 1.1** 所示，一个负载均衡服务器后面挂载若干服务，每个服务都要 N 个节点。由于目前常用的免费负载均衡方案中，nginx 的使用率更广，下面所有的讲解都是以 nginx 为模板。简单起见，我们首先考虑部署在内网中，服务和服务相互调用的情况。

我们假设 `服务1` 的端口为 `8080`，那么配置 `服务1` 的nginx 配置文件可能就是这个样子的：

```nginx
upstream service1 {
    server 192.168.1.2:8080;
    server 192.168.1.3:8080;
    server 192.168.1.4:8080;
}

server {
    listen       8080;

    location ^~ /api/ {
        proxy_pass  http://service1;
        proxy_redirect off;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;    
        proxy_set_header Host $http_host;
        proxy_connect_timeout   120;
        proxy_send_timeout      120;
        proxy_read_timeout      120;
    }
    location / {
        root /opt/dist/;
        index index.html;
    }
}
```

**代码 1.1**

**代码 1.1** 中，访问 http://nginxip:8080/api 开头的链接，都会被反向代理到 `192.168.1.2` `192.168.1.3` `192.168.1.4` 。如果我们机器的节点数变动不大，一切安好，但是随着我们业务量的上升，免不了要增加节点，这时候就必须手动修改 **代码 1.1**，增加所需的节点，然后重启 nginx。如果 `服务1` 变动不频繁，其实也可以忍。但是我们的 nginx 上可是并不仅仅只有一个 `服务1` 的，它还有 `服务2`，未来可预见，还有 `服务3` `服务4`。`服务1` 自己安分守己，但是它无法预见同样跟它一样依赖 nginx 做服务负载均衡的 `服务X` 在未来的某一天不会抽风。

## 2 解决方案

说白了，我们当前的架构中是有类中心化的节点的，我们的 nginx 就是这个中心化的节点。`服务X` 带来的流量激增会伤害到看上去跟它无关的 `服务Y`。`服务 X` 的流量激增，是 `服务X` 导致的，本着谁污染谁治理的原则，怎么也得是 `服务X` 的请求者来解决吧，但是由于每个服务都有多个动态变动的节点，请求者并不知道他们的地址。这个时候，就轮到我们的 `服务发现` 技术闪亮登场了。

`服务发现` 技术解决方案，都会提供一个应用注册中心。服务器端可以将自己的访问地址注册到这个中心里来；然后客户端可以通过监听注册中心中当前服务的变动情况，来动态增加本地维护的服务器列表。

![](/images/register_and_watch.png)

**图 1.2**

这样处在中心节点的负载均衡服务组件不复存在，具体到某个应用的流量激增，也就不会直接影响其他应用。也许你会有疑问，注册中心也是中心节点吧，但是注册中心仅仅用来做应用启动注册和节点变动通知，并发的压力的很小，并不会因为流量的增加也受到影响。况且这个所谓的中心节点，也并不仅仅是说只有一个单节点，而是一个高可用的集群结构。

常见的 `服务发现` 组件有 consul zookeeper etcd 等。本文选用的是 consul ，因为它的使用者比较广泛，且功能比较全面。

> 可以参见这篇文章来对比不同组件之间的差异 http://huhansi.com/2020/04/14/SpringCloud/2020-04-14-004-%E5%90%84%E7%A7%8D%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0%E6%A1%86%E6%9E%B6%E5%AF%B9%E6%AF%94/

新技术栈的引进，必然会带来改造的工作量。对于服务器端来说，改造不大，仅仅是在应用启动的时候增加一个注册步骤即可。这个注册步骤甚至都可以直接做成一个单独的应用程序，随机器启动，这样对于原有项目代码没有侵入性。

> 可以参照笔者写的 [consul-register](https://github.com/yunnysunny/consul-register) 项目来做服务注册。这个项目就是通过单独应用来注册服务，不需要修改原有项目代码。

客户端的改造要复杂些，因为要在客户端本身实现负载均衡算法。

> 客户端负载均衡的实现，可以参见笔者写的 [consul-balancer](https://github.com/yunnysunny/consul-balancer) 项目。

**图 1.2** 适合的是在集群内部服务器之间相互调用的情况，但是一个服务如果是对公网暴漏访问的，情况会大不同。对于公网访问的服务，我们需要提供 https 支持（这里仅仅考虑七层 HTTP 协议，不考虑四层 socket 长连接的情况）。这时候，我们不得不在公网入口处部署一处 nginx 服务器（也可以是其他类似服务器，这里仅仅来 nginx 举例），配置好 SSL 证书，完成 https 的握手。考虑到这个 nginx 是直接和用户建立连接，如果这个 nginx 只有一台机器，而你的用户群体又分布在全国各地，肯定会出现一定概率某个地域的用户访问这台单节点的 nginx 服务速率慢的问题。所以更优化的方案是全国几个重要的地区各自建立入口点 nginx，最终入口点和业务集群之间的网络拓扑结构会是这样。

![](/images/edge_to_cluster.png)

**图 1.3**

**图 1.3** 上面画了三个入口，其通过专线连接到了业务集群，但是这里并没有画入口节点是怎么最终连接到 `服务X` 上的节点的。如果入口节点和集群内部的网络是隔离的，那么入口节点和集群必然要部署一台负载均衡服务器，用来做流量转发。这太负载均衡服务器上绑定两块网卡，一块负责和入口节点通信，一块负责和业务集群通信。

![](/images/edge_to_balancer.png)

**图 1.4**

**图 1.4** 跟 **图 1.3** 相比，多了一个负载均衡服务器。嗯？但是这个样子的话，岂不是又走回了我们的老路。而且说好的用服务发现呢，这也没发现啊。对于后者我们可以使用 consul-template 这个工具，读取 consul 的服务配置，动态生成 nginx 配置文件，然后重载 nginx 来达到服务发现的目的。

![](/images/consul_template_in_cluster.png)

**图 1.5**

使用了 consul-template 之后，我们利用上了服务发现技术栈，但是这里面的负载均衡服务器依然是一个中心节点。虽然我们可以通过增加负载均衡服务器节点个数来承载更高的流量，但是一来这样子会增加更多的机器开销，二来增加负载均衡服务节点，同样要修改入口节点配置。

如果入口节点和业务集群的网络是打通的，问题会好办很多。入口点直接读取注册中心的变动，动态更改本地服务器列表即可。

![](/images/private_line_with_service_discovery.png)

**图 1.6**

**图 1.6** 简化一下入口点，仅仅画出了一处作为代表。入口节点直接读取注册中心，如果服务有节点变动，便会及时更新本地配置文件，可以直接访问到 `服务X` 上的节点。

使用 consul-template 其实隐藏了一个问题，服务器上的节点在出现异常的时候，进程可能崩溃退出，注册到注册中心上的节点就会发送变动，consul-template 就会更新入口节点上的配置文件，触发 nginx 的重启。但是 nginx 的重启过程在高并发的情况下，会导致不必要的延迟出现，对于低延迟应用来说是个很大的打击。

> 关于重载 nginx 导致的性能问题，可以参见官方的这篇博文 https://www.nginx.com/blog/using-nginx-plus-to-reduce-the-frequency-of-configuration-reloads/

解决这个问题，我们可以依靠 openresty 这个 nginx 插件，它支持通过 lua 来进行编程，可以拦截 nginx 处理 http 请求的各个阶段。依靠这个特性，我们可以定时拉取各个服务的节点配置信息到 openresty 中存储起来，在客户端触发请求的时候，直接转发请求到具体的某一个节点上面。

> 大家可以参考笔者的项目 [yunnysunny/resty-gate](https://github.com/yunnysunny/resty-gate)，它实现了通过 openresty 动态拉取 consul 上的服务列表并基于此进行负载均衡的功能。

![](/images/consul_with_openresty.png)

**图 1.7**









