---
attachments: [Clipboard_2021-05-18-21-34-23.png, Clipboard_2021-05-18-21-39-17.png, Clipboard_2021-05-18-21-48-38.png, Clipboard_2021-05-18-23-05-33.png]
title: Nginx 基础知识
created: '2021-05-18T11:30:07.965Z'
modified: '2021-05-18T15:20:59.255Z'
---

# Nginx 基础知识

## Nginx介绍
Nginx是俄罗斯人Igor Sysoev开发的**轻量级Web服务器**，不仅仅是一个高性能的HTTPP和反向代理服务器，同时也是一个IMAP/POP3/SMTP服务器。它是以**事件驱动**的方式编写，所以有非常好的性能，同时也是一个非常高效的反向代理、负载平衡服务器。在性能上，Nginx占用很少的系统资源，能支持更多的并发连接，达到更高的访问效率；在功能上，Nginx是优秀的代理服务器和负载均衡服务器；在安装配置上，Nginx安装简单、配置灵活。Nginx支持热部署，启动速度特别快，还可以在不间断服务的情况下对软件版本或配置进行升级，即使运行数月也无需重新启动。在微服务的体系之下，Nginx正在被越来越多的项目采用作为网关来使用，配合Lua做限流、熔断等控制。

## Nginx特点
> - 轻量级Web服务器，结构简单，开源免费
> - 事件驱动，异步非阻塞处理，多路复用IO
> - 零拷贝sendfile支持
> - 七层负载，天然支持HTTP代理
> - 多模块高拓展
> - 多进程模型高可靠性，一个主进程，多个工作进程
> - 热部署
>
> 可用作**HTTP正反向代理**，**负载均衡**和**动静分离**。

Nginx示意：
![Nginx示意](@attachment/Clipboard_2021-05-18-23-05-33.png)

### Nginx 正向代理
当客户端无法直接与服务器建立连接时(没错，就是你GFW的锅)。我们就需要一个正向代理来为我们搭建连接建立的桥梁。

正向代理示意：
![正向代理示意](@attachment/Clipboard_2021-05-18-21-34-23.png)
> 客户端明确的知道要访问的是代理服务器。真实的目标交由代理服务器去连接。

### Nginx 反向代理
看起来好像和正向代理的访问模式一样，都是客户端连接Nginx，Nginx连接到服务器。

反向代理示意：
![反向代理示意](@attachment/Clipboard_2021-05-18-21-39-17.png)
> 客户端真正期望连接的`www.ab.com`实际是指服务器A或服务器B。但是实际连接的却是Nginx，真实的目标服务器被隐藏了。

最大并发数：worker_connection * worker_processes / 4。占用2个连接，客户端-worker，worker-服务器。
> worker_connection：一个Worker的最大连接数
> 
> worker_processes：工作进程数
>
> 因为一个请求要占用2个或4个连接数。

### Nginx 负载均衡
就是对请求进行一个调度，默认的策略是轮询调度，均匀的分配给每个服务器。

### Nginx 动静分离
把一些css，js，图片等脚本静态资源内容放在另一台服务器上，类似于CDN一般，这样就可以把加载时的资源分离开来，实现单独的负载均衡等。

动静分离示意：
![动静分离示意](@attachment/Clipboard_2021-05-18-21-48-38.png)
> 不分离的话，一次请求既要包括大量IO操作，同时动态请求还包含了大量CPU计算操作，对于服务器的压力较大。而且静态资源设置了过期时间后，一定时间内的请求都可以无需加载。

## Nginx 事件模型
Nginx支持5种连接处理方法，可以通过`use`来指定：
1. select：标准方法。如果当前平台没有更高效的方法，属于编译时默认的方法，可以使用配置参数
`--with-select_module`和`--without-select_module`
2. poll：标准方法。 如果当前平台没有更有效的方法，它是编译时默认的方法。你可以使用配置参数
`--with-poll_module`和`--without-poll_module`
3. kqueue：高效的方法，使用于 FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0 和 MacOS X. 使用**双处理器的MacOS X系统使用kqueue可能会造成内核崩溃**。
4. epoll：高效的方法，使用于Linux内核2.6版本及以后的系统。在某些发行版本中，如SuSE 8.2, 有让2.4版本的内核支持epoll的补丁。
5. /dev/poll：高效的方法，使用于 Solaris 7 11/99+, HP/UX 11.22+ (eventport), IRIX 6.5.15+ 和 Tru64 UNIX 5.1A+.
6. eventpoll：高效的方法，使用于 Solaris 10.为了防止出现内核崩溃的问题，需要安装安全补丁。

## Nginx KeepAlived高可用方案
参考LVS KeepAlived方案，配合官方文档，大同小异。毕竟KeepAlived也是一个单独的应用存在。提供了对负载均衡器的通用配置体系和监控系统。但是由于是单独的应用，所以还需要zookeeper来保证可靠性。
