---
title: Kafka基础之简单搭建
created: '2021-05-11T11:44:44.428Z'
modified: '2021-05-11T15:05:54.091Z'
---

# Kafka基础之简单搭建

## 环境准备
- Linux
- JDK1.8+
- Zookeeper [下载地址](https://downloads.apache.org/zookeeper/)
- Kafka [下载地址](http://kafka.apache.org/downloads)

## 单机Kafka搭建
注意事项：
- 配置主机名和IP映射
- 关闭防火墙&防火墙开机自启动

### 配置基础环境
1. 安装JDK，并配置好环境变量，最好先确认是否有Java，可以把老版本的卸载。
> rpm包安装：`rpm -ivh jdk-8u291-linux-aarch64.rpm`
> tar.gz包安装：`tar -zxvf jdk-8u291-linux-aarch64.tar.gz`
> Java的安装目录，假设是`/usr/java/jdk-8u291`这样。
>
> 环境变量可以在当前用户级别配置即`.bashrc`文件，也可以去全局配置`/etc/profile`下配置。
> 这里以.bashrc为例：
> `JAVA_HOME=/usr/java/jdk-8u291`
> `PATH=$PATH:$JAVA_HOME/bin`
> `export JAVA_HOME`
> `export PPATH`
>
> 最后要让改的配置生效`source .bashrc`

2. 修改配置下的主机名和IP映射
> 主机名在`/etc/sysconfig/network`下。
> 修改主机名：`HOSTNAME=CentOS`主机名随便取，这里叫CentOS。
> IP映射在`/etc/hosts`下。
> 修改IP映射：`192.168.181.128 CentOS`，跟windows一样的配法。

3. 关闭防火墙和防火墙的自启。
> 先使用`systemctl firewalld status`可以查看防火墙状态。
> 直接`systemctl firewalld stop`可以关闭防火墙。
> 最后`systemctl disabled firewalld`可以关闭防火墙的开机自启。

4. 配置zookeeper。
> 下载的tar.gz包先解压：`tar -zxvf zookeeper-3.6.3.tar.gz`
> Zookeeper的安装目录，假设是`/usr/zookeeper-3.6.3`这样。
> 
> zookeeper的模板配置文件：`/usr/zookeeper-3.6.3/conf/zoo_sample.cfg`
> 复制一份模板配置文件在同目录下：zoo.cfg。
> 修改zoo.cfg的选项：`dataDir=/root/zkdata`变更数据目录。
> 使用`./usr/zookeeper-3.6.3/bin/zkServer.sh`可以启动zk了。
> 例：`./bin/zkServer.sh start conf/zoo.cfg`启动zk。

5. 配置Kafka
> 下载的tar.gz包先解压：`tar -zxvf kafka_2.11-2.6.2.tgz`
> kafka的安装目录，假设是`/usr/kafka_2.11-2.6.2`这样。
>
> kafka的模板配置文件：`/usr/kafka_2.11-2.6.2/conf/server.properties`
> 修改server.properties选项：`listeners=PLAINTEXT://CentOS:9092`socket连接地址，这里**最好填主机名，不要IP**。
> 修改server.properties选项：`log.dirs=/usr/kafka-logs`消息日志存储地址。
> 修改server.properties选项：`zookeeper.connect=CentOS:2181`zk连接配置。
> 使用`./usr/kafka_2.11-2.6.2/bin/kafka-server-start.sh`可以启动kafka了。
> 例：`./bin/kafka-server-start.sh -daemon config/server.properties`后台启动。

### 配置Topic
使用Kafka的`kafka-topics.sh`脚本来执行topic，分区，副本因子等配置。
> 连接上Broker服务器：`./bin/kafka-topics.sh --bootstrap-server CentOS:9092 --create --topic topic01 --partitions 2 --replication-factor 1`，创建了一个topic为topic01的主题，该topic下配置了2个分区日志数，同时为每一个分区日志配置了一个副本，CentOS作为Broker服务器主分区的Leader。
>
> 消费者开启：`./bin/kafka-console-consumer.sh --bootstrap-server CentOS:9092 --topic topic01 --group group1`，创建了一个消费者实例，消费topic为topic01这个主题下的消息，位于group1这个消费者组下，连接Broker服务器CentOS，这个时候group1只有一个消费者实例，所以会同时消费2个分区的数据。
>
> 生产者开启：`./bin/kafka-console-producer.sh --broker-list CentOS:9092 --topic topic01`，创建了一个生产者，将要想topic为topic01这个主题下投递消息，并连接当前主题的Broker服务器CentOS。

## 集群Kafka搭建
注意事项(包含以上)：
- 配置同步时钟：`ntpdate cn.pool.ntp.org`，`ntpdate ntp[1-7].aliyun.com`[]里选填一个值就行。

### 集群间配置
1. 在hosts文件中配置上多个服务器的IP映射
> `192.168.181.128 CentOSA`,`192.168.181.129 CentOSB`,`192.168.181.130 CentOSC`等几台都可，这里是3台。
> 同理集群中的其他服务器也需要这么配置。

2. 配置zookeeper的配置文件zoo.cfg
> 在zoo.cfg后新增：`server.1=CentOSA:2888:3888`，`server.2=CentOSA:2888:3888`，`server.3=CentOSA:2888:3888`等服务器的地址。
> 同理*3
> 2181端口：对cline端提供服务
> 2888端口：集群内机器通讯使用（Leader监听此端口）
> 3888端口：选举leader使用

3. 配置Kafka集群
> 修改server.properties选项：`zookeeper.connect=CentOSA:2181,CentOSB:2181,CentOSC:2181`zk服务器连接配置，都配置上。
> 同理*3
> 修改server.properties选项：`broker.id=1`，`broker.id=1`，`broker.id=2`几个Broker的ID需要**唯一**配置。

### 集群间配置topic
和单机版差不多的使用方式。多多利用命令--help，啥都有。还有官方文档可以查阅。

### 小结
试了一下搭建，主要理解一下大致的运作，试试就好。

