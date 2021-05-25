---
attachments: [Clipboard_2021-05-18-21-34-23.png, Clipboard_2021-05-18-21-39-17.png, Clipboard_2021-05-18-21-48-38.png, Clipboard_2021-05-18-23-05-33.png]
title: Nginx 基础知识
created: '2021-05-18T11:30:07.965Z'
modified: '2021-05-19T14:46:06.815Z'
---

# Nginx 基础知识

## Nginx介绍
Nginx是俄罗斯人Igor Sysoev开发的**轻量级Web服务器**，不仅仅是一个高性能的HTTPP和反向代理服务器，同时也是一个IMAP/POP3/SMTP服务器。它是以**事件驱动**的方式编写，所以有非常好的性能，同时也是一个非常高效的反向代理、负载平衡服务器。在性能上，Nginx占用很少的系统资源，能支持更多的并发连接，达到更高的访问效率；在功能上，Nginx是优秀的代理服务器和负载均衡服务器；在安装配置上，Nginx安装简单、配置灵活。Nginx支持热部署，启动速度特别快，还可以在不间断服务的情况下对软件版本或配置进行升级，即使运行数月也无需重新启动。在微服务的体系之下，Nginx正在被越来越多的项目采用作为网关来使用，配合Lua做限流、熔断等控制。

## Nginx 优点
> - 跨平台：Nginx可以在大多数UNix like OS平台运行，也有windows的移植版。
> - 配置非常简单：配置风格与程序开发类似，易于理解。
> - 非阻塞，高并发连接：官方测试可以支撑5万并发连接，实际中可以达到2~3万。
> - 事件驱动：如果在linux上，默认采用epoll模型，支持更大的连接数。
> - 热部署能力：一个主进程，多个工作进程，可以直接修改配置生效而无需重启。
> - 内存消耗小：处理大并发的请求内存消耗非常小，在3万并发连接中，10个Nginx才消耗150M内存。
> - 内置的健康检测：Nginx的某台后端服务器宕机，不会影响前端访问。(其他使用KeepAlived也可达到此效果)
> - 节省带宽：支持GZIP压缩，可以添加浏览器本地缓存的请求头。
>
> 可用作**HTTP正反向代理**，**负载均衡**和**动静分离**。

## Nginx 缺点
> - Nginx仅能支持Http，Https和SMTP协议，在适用范围上小。
> - 内置的健康检测仅能支持端口，不支持url检测。不用ip_hash负载无法支持对session的保持。(session共享也行)

Nginx示意：
![Nginx示意](@attachment/Clipboard_2021-05-18-23-05-33.png)
> Master进程负责监控和通知Worker进程处理连接，通过accept_mutex信号量来控制Worker竞争连接。(对比Netty BossGroup和WorkerGroup)
> 
> 一个请求只能被一个Worker处理，一个Worker只有一个主线程，所以同时也只能处理一个请求。

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

upstream模块下负载均衡的配置策略：
> - 轮询(默认)：每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动踢除。
> - weight：指定轮询权重，weight和访问比例成正比，值越大，请求越多。
> - ip_hash：每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
> - fair(第三方)：按后端服务器的响应时间来进行分配，响应时间短的优先分配。
> - url_hash(第三方)：按访问url到hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。

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

## Nginx 优化

### 编译安装过程优化
编译Nginx的时候，默认以DEBUG模式运行，该模式下会插入很多跟踪与assert之类的信息，编译完成后，一个Nginx有好几M大小，而如果取消DEBUG模式，则可以减少占用空间，大概几百K的大小，修改方式如下：
找到源码目录下`/auto/cc/gcc`文件，找到
`# debug`
`CFLAGS="$CFLAGS -g"`
删除或注释掉这2行即可取消debug模式编译。

### 为指定CPU进行类型编译优化
编译Nginx的时候，默认的GCC编译参数是"-O"，要优化GCC编译，可以使用以下两个参数：
`--with-cc-opt='-O3'`
`--with-cpu-opt=CPU`：CPU即为特定的CPU编译，有效值包括：
> - pentium
> - pentiumpro
> - pentium3
> - pentium4
> - athlon
> - opteron
> - amd64
> - sparc32
> - sparc64
> - ppc64
>
> 确认CPU类型：`cat /proc/cpuinfo | grep "model name"`

### 利用TCMalloc库
TCMalloc的全称为Thread-Caching Malloc，是谷歌开源工具google-perftools中的一个成员，比标准glibc库的内存分配效率和速度上要快很多。提高了服务器高并发下的性能，降低负载。

### TCP内核参数
可以定制一些TCP内核参数，来让Nginx选择对连接的处理方式。
具体配置参数在TCP/IP内介绍。



