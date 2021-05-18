---
title: Nginx 简单配置
created: '2021-05-18T13:12:02.774Z'
modified: '2021-05-18T15:09:39.939Z'
---

# Nginx 简单配置

## 配置介绍
Nginx多模块下，完全依靠配置文件来进行配置。
[Nginx配置中文文档](https://www.nginx.cn/doc/)

### Location 映射配置
`location [=|~|~*|^~] /uri/ { … }`
> - `=` 开头表示精确匹配
> - `^~` 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。以xx开头
> - `~` 开头表示区分大小写的正则匹配，以xx结尾
> - `~*` 开头表示不区分大小写的正则匹配，以xx结尾
> - `!~` 和`!~*` 分别为区分大小写不匹配及不区分大小写不匹配的正则
> - `/` 通用匹配，任何请求都会匹配到。

## 配置文件解析
```json
# 全局块
# user nobody
worker_processes  1; # 工作线程，数量越大，可以支持的并发数越多，但是受硬件制约。

# error_log logs/error.log;
# error_log logs/error.log notice;
# error_log logs/error.log info;

#pid        logs/nginx.pid;

# EVENT事件块
events {
  worker_connections  1024; # 单Worker支持的最大连接数。
}

# HTTP全局块 需要配置的块
http {
  include mime.types;
  default_type  application/octet-stream;
  sendfile  on; # 启用零拷贝
  access_log  /var/log/nginx/access.log; # 设定日志格式
  keepalive_timeout 65; # 长连接保活时间

  server {
    listen  80; # 监听的端口号
    server_name localhost; # 主机地址

    location / { # 代理的请求格式 /=所有，可以用正则表达式配置规则
      alias /data/html/; # 目录别名，查找资源时直接按此目录查找，必须以/结尾
      root  /data/html/; # 目录根名，查找资源时将这个拼接在资源路径前，可选/结尾
      index index.html  index.htm; # 默认请求地址
    }

    error_page 500 502 503 504 /50x.html; # 异常界面，HTTP状态码路由
    location = /50x.html {
      root  html;
    }
  }
}
```

## 正向代理配置
```json
http {
  include mime.types;
  default_type  application/octet-stream;

  keepalive_timeout 65; # 长连接保活时间
  server {
    # 端口
    listen       8080;
    # 地址
    server_name  localhost;
    # DNS解析地址
    resolver 8.8.8.8;
    # 代理参数
    location / {
        # $http_host就是我们要访问的主机名
        # $request_uri就是我们后面所加的参数
        proxy_pass http://$http_host$request_uri;
    }
  }
}
```

## 反向代理配置
主要修改HTTP块
```json
http {
  include mime.types;
  default_type  application/octet-stream;
  keepalive_timeout 65; # 长连接保活时间

  server {
    listen  80; # 监听的端口号
    server_name 192.168.180.181; # 主机地址

    location ~ /A/ { # 代理的请求格式 以A为根目录结构的请求
      root  html; # 目录位置
      proxy_pass  http://192.168.180.182:8080; # HTTP代理的服务器A地址
      index index.html  index.htm;
    }

    location ~ /B/ { # 代理的请求格式 以B为根目录结构的请求
      root  html;
      proxy_pass  http://192.168.180.183:8080; # HTTP代理的服务器B地址
      index index.html  index.htm;
    }

    error_page  500 502 503 504 /50x.html;
    location = /50x.html {
      root  html;
    }
  }
}
```

## 负载均衡配置
```json
#设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
 
    #设定mime类型,类型由mime.type文件定义
    include            /etc/nginx/mime.types;
    default_type    application/octet-stream;
 
    #设定日志格式
    access_log        /var/log/nginx/access.log;
 
    #省略上文有的一些配置节点
    #。。。。。。。。。。
 
    #设定负载均衡的服务器列表
    upstream mysvr {
        #weigth参数表示权值，权值越高被分配到的几率越大
        server 192.168.8.1x:3128  weight=5;
        #本机上的Squid开启3128端口,不是必须要squid
        server 192.168.8.2x:80    weight=1;
        server 192.168.8.3x:80    weight=6;
    }
        
    upstream mysvr2 {
        #weigth参数表示权值，权值越高被分配到的几率越大
        server 192.168.8.x:80    weight=1;
        server 192.168.8.x:80    weight=6;
    }
 
    #第一个虚拟服务器
    server {
        #侦听192.168.8.x的80端口
        listen             80;
        server_name    192.168.8.x;
 
        #对aspx后缀的进行负载均衡请求
        location ~ .*.aspx$ {
            #定义服务器的默认网站根目录位置
            root     /root; 
            #定义首页索引文件的名称
            index index.php index.html index.htm;
            
            #请求转向mysvr 定义的服务器列表
            proxy_pass    http://mysvr ;
 
            #以下是一些反向代理的配置可删除.
 
            proxy_redirect off;
 
            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 
            #允许客户端请求的最大单文件字节数
            client_max_body_size 10m; 
 
            #缓冲区代理缓冲用户端请求的最大字节数，
            client_body_buffer_size 128k;
 
            #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_connect_timeout 90;
 
            #连接成功后，后端服务器响应时间(代理接收超时)
            proxy_read_timeout 90;
 
            #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffer_size 4k;
 
            #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
            proxy_buffers 4 32k;
 
            #高负荷下缓冲大小（proxy_buffers*2）
            proxy_busy_buffers_size 64k; 
 
            #设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy_temp_file_write_size 64k;    
 
        }
    }
}
```

## 动静分离配置
```json
http {
  include mime.types;
  default_type  application/octet-stream;
  keepalive_timeout 65; # 长连接保活时间

  server {
    listen  80; # 监听的端口号
    server_name 192.168.180.181; # 主机地址

    location / {  
      root   e:wwwroot;  
      index  index.html;  
    }

    # 所有静态请求都由nginx处理，存放目录为html  
    location ~ .(gif|jpg|jpeg|png|bmp|swf|css|js)$ {  
        root    /data/asset/; # 资源目录
    }  

    # 所有动态请求都转发给tomcat处理  
    location ~ .(jsp|do)$ {  
        proxy_pass  http://192.168.180.182:8080;  
    }

    error_page  500 502 503 504 /50x.html;
    location = /50x.html {
      root  html;
    }
  }
}
```

## 小结
1. 工作进程配置越大性能越好吗？
> 并不，合适最好，根据CPU核数选择，相等即可。
2. 代理怎么配置？
> 主要是HTTP块的配置，看官方文档。

