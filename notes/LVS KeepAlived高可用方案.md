---
attachments: [Clipboard_2021-05-17-23-37-52.png, Clipboard_2021-05-17-23-55-04.png]
title: LVS KeepAlived高可用方案
created: '2021-05-17T12:13:59.600Z'
modified: '2021-05-17T15:57:19.434Z'
---

# LVS KeepAlived高可用方案

## KeepAlived介绍
Keepalived是Linux下一个轻量级别的高可用解决方案。高可用(High Avalilability,HA)，其实两种不同的含义：广义来讲，是指整个系统的高可用行，狭义的来讲就是之主机的冗余和接管，它与HeartBeat RoseHA 实现相同类似的功能，都可以实现服务或者网络的高可用，但是又有差别，HeartBeat是一个专业的、功能完善的高可用软件，它提供了HA 软件所需的基本功能，比如：心跳检测、资源接管，检测集群中的服务，在集群节点转移共享IP地址的所有者等等。HeartBeat功能强大，但是部署和使用相对比较麻烦，与HeartBeat相比，Keepalived主要是通过虚拟路由冗余来实现高可用功能，虽然它没有HeartBeat功能强大，但是Keepalived部署和使用非常的简单，所有配置只需要一个配置文件即可以完成。

KeepAlived起初是为了LVS而设计，提供配置文件，后期通过`vrrp`协议(Vritrual Router Redundancy Protocol,虚拟路由冗余协议)允许与nginx，HaProxy等配合使用。KeepAlived有2种配置模式，**抢占式**和**非抢占式**。

## LVS主备模式(抢占式)
抢占式就是主机一旦挂掉，备机立刻会抢主机的位置，自身成为新主机。

主备模式演示：
![主备模式演示](@attachment/Clipboard_2021-05-17-23-37-52.png)

主机备机配置文件：
``` json
vrrp_instance VI_1 {
  state MASTER          # 主机：MASTER，备机：BACKUP。其他一样。
  interface eth0        # 需要监测的网卡
  virtual_router_id 21  # 让主机和备机在同一个虚拟路由下，ID必须相同
  priority 100          # 优先级，决定选择的优先级，越大优先级越高
  advert_int 1          # 心跳间隔时间，单位秒
  authentication {
    auth_type PASS      # 认证
    auth_pass 1234      # 密码
  }
  vitual_ipaddress {    # 只有主机有VIP，备机没有，路由
    192.168.180.233/24 dev eth0 label eth0:1 # LVS的VIP/NetMask dev 网卡名 label 网卡名：子接口号
  }
}
virtual_server 192.168.180.233 80 { # LVS的VIP + 实际端口
  delay_loop 6            # 
  lb_algo rr              # 负载调度算法，rr=轮询
  lb_kind DR              # 工作模式
  nat_mask 255.255.255.0  # 掩码
  persistence_timeout 50  # 负载连接维护时长，单位秒
  protocol TCP            # 传输协议
  real_server 192.168.180.182 80 { # 真实服务器的IP与端口，可以配置多个真实服务器。
    weight 1              # 权重
    HTTP_GET {            # 健康检查形式，HTTP_GET，SSL_GET
      url {               # 访问确认地址，可以配置多个。
        path /            # 访问地址，此处为访问服务器根地址。
        status_code 200   # 如何判定成功，此处为响应状态码为200时认为ok。
      }
      connect_timeout 3   # 连接超时时间
      nb_get_retry 3      
      delay_before_retry 3
    }
  }
}
```

存在**不可靠**的问题：
> 如果KeepAlived应用**异常退出**，导致主机的VIP可能没有被收回，LVS映射MAC地址也没有收回。然后备机发现无法联通主机，备机自动抢占为主机，但是这时局域网内就存在了2个相同的暴露出去的VIP。数据包发送就不确定了，不可靠。

## LVS集群模式(非抢占式)
配置文件上区别不大，多了一个属性：
`nopreempt`：标识为非抢占式，也就是投票选择主机，公平模式。

引入zookeeper来保证KeepAlived高可用应用集群的可靠性。

## KeepAlived脑裂现象
脑裂现象一般发生在抢占式中。由于某些原因，导致两台KeepAlived高可用服务器在指定时间内，无法检测到对方的存货心跳信息，从而导致互相抢占对方的资源和服务所有权，然后两台却还都存活中。

脑裂示意图：
![脑裂示意图](@attachment/Clipboard_2021-05-17-23-55-04.png)

脑裂发生的原因：
> - 开启了防火墙，限制了对方的访问
> - 网线故障，局域网不连通
> - 心跳时间极短，这个应该不会

