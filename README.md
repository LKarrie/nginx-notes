# NGINX Notes

* [NGINX Notes](#nginx-notes)
  * [安装](#安装)
    * [初始化工作环境](#初始化工作环境)
    * [准备源码包](#准备源码包)
    * [编译安装](#编译安装)
  * [运维](#运维)
    * [热升级/重启](#热升级重启)
    * [日志管理](#日志管理)
    * [开机自启](#开机自启)
    * [监控](#监控)
    * [高可用](#高可用)
  * [常用技巧和配置](#常用技巧和配置)



## 安装

实践使用的版本（社区） 1.18.0 1.20.2

比较推荐的三方模块

* [vts-用于监控NGINX流量和状态](https://github.com/vozlt/nginx-module-vts)

* [proxy_connect-正向代理解决方案](https://github.com/chobits/ngx_http_proxy_connect_module)

* [dynamic_upstream-动态更新upstream server](https://github.com/cubicdaiya/ngx_dynamic_upstream)



### 初始化工作环境

```markdown
yum -y install gcc gcc-c++ libtool make patch   
yum -y install pcre pcre-devel          
yum -y install zlib zlib-devel
yum -y install openssl openssl-devel
```

**注意**：openssl 版本 和TLS 版本有关

如果你的nginx需要支持 TLSv1.3 openssl需要使用较新的版本（利例如 1.1.1g）



### 准备源码包

从官网下载对应版本 [nginx: download](https://nginx.org/en/download.html)

如有需要的三方模块 前往对应 github页面下载 

如有需要，下载相关社区模组源码包



个人比较喜欢的NGINX编译目录

module 存放 三方模块源码

module_tar 存放 三方模块源码压缩包

```terminal
[root@nginx nginx_build]# pwd
/app/nginx_build
[root@nginx nginx_build]# tree -L 2
.
├── module
│   ├── nginx-module-vts
│   ├── ngx_dynamic_upstream
│   ├── ngx_healthcheck_module
│   ├── ngx_http_proxy_connect_module
│   └── read-request-body-nginx-module
├── module_tar
│   ├── nginx-module-vts-0.2.1.tar.gz
│   ├── nginx-rewrite-request-body-module-master.zip
│   ├── ngx_dynamic_upstream-0.1.6.zip
│   ├── ngx_healthcheck_module-master.zip
│   ├── ngx_http_proxy_connect_module-0.0.3.tar.gz
│   └── read-request-body-nginx-module-master.zip
├── nginx-1.18.0
│   ├── auto
│   ├── CHANGES
│   ├── CHANGES.ru
│   ├── conf
│   ├── configure
│   ├── contrib
│   ├── html
│   ├── LICENSE
│   ├── Makefile
│   ├── man
│   ├── objs
│   ├── README
│   └── src
├── nginx-1.18.0.tar.gz
├── nginx-1.20.2
│   ├── auto
│   ├── CHANGES
│   ├── CHANGES.ru
│   ├── conf
│   ├── configure
│   ├── contrib
│   ├── html
│   ├── LICENSE
│   ├── Makefile
│   ├── man
│   ├── objs
│   ├── README
│   └── src
└── nginx-1.20.2.tar.gz
```



### 编译安装

> 常用官方模块 + 三方模块vts

```bash
cd /app/nginx_build
tar -zxvf nginx-1.20.2.tar.gz
cd nginx-1.20.2

./configure --prefix=/app/nginx/nginx-1.18.0 --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --with-http_sub_module --add-module=/app/nginx_build/module/nginx-module-vts

make
make intall
```



> 常用官方模块 + 三方模块vts + 三方模块proxy_connect

```bash
cd /app/nginx_build
tar -zxvf nginx-1.20.2.tar.gz
cd nginx-1.20.2
patch -p1 < /app/nginx_build/module/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_1018.patch

./configure --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --with-http_sub_module --add-module=/app/nginx_build/module/nginx-module-vts --add-module=/app/nginx_build/module/ngx_http_proxy_connect_module

make
make intall
```



> 常用官方模块 + 编译NGINX Openssl版本升级(支持TLS 1.3) + 三方模块vts

openSSL升级

```bash
cd /app
wget https://www.openssl.org/source/openssl-1.1.1g.tar.gz
tar -xvf openssl-1.1.1g.tar.gz
cd openssl-1.1.1g
./config shared --openssldir=/usr/local/openssl --prefix=/usr/local/openssl
make && make install

echo "/usr/local/lib64/" >> /etc/ld.so.conf
ldconfig

mv /usr/bin/openssl /usr/bin/openssl.old
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/openssl/include/openssl /usr/include/openssl
echo "/usr/local/openssl/lib" >> /etc/ld.so.conf

ldconfig -v
```

编译指定openSSL

```bash
cd /app/nginx_build
tar -zxvf nginx-1.20.2.tar.gz
cd nginx-1.20.2

./configure --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --with-http_sub_module --with-openssl=/app/openssl/openssl-1.1.1g --add-module=/app/nginx_build/module/nginx-module-vts

make
make intall
```

可以查看编译后版本验证相关信息（built with OpenSSL xxx）

```terminal
[root@lkarrie /]# /app/nginx/sbin/nginx -V
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.1.1g  21 Apr 2020
TLS SNI support enabled
configure arguments: --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --with-http_sub_module --with-openssl=/app/openssl/openssl-1.1.1g --add-module=/app/nginx_build/module/nginx-module-vts
```



个人比较喜欢的NGINX安装目录

conf/cert 存放证书

script 存放keepalived探活脚本 或 一些定时任务脚本

```terminal
[root@lkarrie nginx]# pwd
/app/nginx
[root@lkarrie nginx]# tree -L 2
.
├── client_body_temp
├── conf
│   ├── cert
│   ├── conf.d
│   ├── fastcgi.conf
│   ├── fastcgi.conf.default
│   ├── fastcgi_params
│   ├── fastcgi_params.default
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types
│   ├── mime.types.default
│   ├── nginx.conf
│   ├── nginx.conf.default
│   ├── scgi_params
│   ├── scgi_params.default
│   ├── uwsgi_params
│   ├── uwsgi_params.default
│   └── win-utf
├── fastcgi_temp
├── html
│   ├── 50x.html
│   └── index.html
├── logs
│   ├── error.log
│   └── nginx.pid
├── proxy_temp
├── sbin
│   └── nginx
├── scgi_temp
├── script
└── uwsgi_temp
```



## 运维

记录一些常用的技巧和操作



### 热升级/重启

热升级或者重启的使用场景：

* NGINX版本升级（替换可执行文件）时，保证流量的同时进行升级

* 在**开源社区版本**中NGINX会对本地DNS进行缓存，没有配置resolver的server，在虚机DNS切换后，NGINX并不会切换，旧的DNS不可用则会影响当前请求解析，需要将NGINX进程重启，重新缓存本地DNS配置（reload无效）

**注**：七层代理可以通过配置 resolver 加set 变量的形式保证代理域名的实际地址最新，而四层代理如果涉及代理地址为域名，虚机DNS切换，必须要重启NGINX

热升级具体操作如下

```markdown
# 进入nginx安装目录下的sbin目录
cd /app/nginx/sbin
# 备份二进制文件
cp -f nginx nginx.old
# 新二进制文件替换旧二进制文件
cp -f /app/nginx_build/nginx-1.20.2/objs/nginx .

# 执行升级和优雅关闭旧worker进程
kill -USR2 69168  
kill -WINCH 69168
        
# 确认升级后有无异常
# 如果有异常执行 故障回滚
# 恢复备份nginx二进制
# mv nginx.old nginx
# 唤醒旧master进程 创建正常worker进程
# kill -HUP 69168
# 优雅退出问题master进程
# kill -QUIT 77179

# 确认无误 无需回退
# 优雅退出旧master进程
kill -QUIT 69168
```

一点演示

```terminal
# 演示
# 升级前nginx版本 编译了nginx_upstream_check_module
[nginx@lkarrie sbin]$ ./nginx -V
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --add-module=/app/nginx_build/module/nginx-module-vts-0.2.1 --add-module=/app/nginx_build/module/nginx_upstream_check_module

# 备份旧nginx二进制文件
[nginx@lkarrie sbin]$ cp -f nginx nginx.old
[nginx@lkarrie sbin]$ ls
nginx  nginx.commonbk  nginx.hpcbk  nginx.old

# 查看旧nginx进程号 
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   76801  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76802  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76803  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76804  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76866  53248  0 05:12 pts/1    00:00:00 grep --color=auto nginx           

# 新nginx二进制文件 覆盖旧nginx二进制文件
[nginx@lkarrie sbin]$ mv /app/nginx_build/nginx-1.20.2/objs/nginx .
[nginx@lkarrie sbin]$ ls
nginx  nginx.commonbk  nginx.hpcbk  nginx.old

# 确认已经覆盖
[nginx@lkarrie sbin]$ ls -al
total 31940
drwxrwxr-x.  2 nginx nginx      77 Nov  7 05:13 .
drwxrwxr-x. 11 nginx nginx     151 Jul 13 19:43 ..
-rwxrwxr-x   1 nginx nginx 8212232 Nov  7 04:56 nginx
-rwxr-xr-x   1 nginx nginx 8087592 Oct 24 21:59 nginx.commonbk
-rwxr-xr-x   1 nginx nginx 8188856 Oct 24 22:00 nginx.hpcbk
-rwxrwxr-x   1 nginx nginx 8209384 Nov  7 05:03 nginx.old

# 没有执行新二进制文件 所以进程pid无变化
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   76801  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76802  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76803  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76804  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76959  53248  0 05:13 pts/1    00:00:00 grep --color=auto nginx

# 执行升级nginx二进制文件 USR2信号
[nginx@lkarrie sbin]$ kill -USR2 69168

# 启动新的nginx master worker进程
# 此时新旧nginx worker进程将同时处理请求
# 如果此时没有新的 master worker进程启动说明nginx conf配置存在异常 无法启动nginx
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   76801  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76802  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76803  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76804  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77181  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77182  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77183  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77192  53248  0 05:16 pts/1    00:00:00 grep --color=auto nginx

# 优雅关闭旧nginx工作进程
# 有可能存在无法停止的旧worker进程 原因是流量较多或存在四层代理NGINX无法主动终止 
# 可以通过设置 worker_shutdown_timeout 在固定时间后强制终止旧worker进程
# 长时间无法停止可以尝试 kill -9 旧worker进程

# 此时只有新nginx worker进程处理请求
[nginx@lkarrie sbin]$ kill -WINCH 69168
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77181  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77182  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77183  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77413  53248  0 05:18 pts/1    00:00:00 grep --color=auto nginx

# 验证更新 nginx_upstream_check_module去除 新增ngx_dynamic_upstream
[nginx@lkarrie sbin]$ ./nginx -V
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --add-module=/app/nginx_build/module/nginx-module-vts --add-module=/app/nginx_build/module/ngx_dynamic_upstream

# 运行一段时间新nginx进程 业务无明显抖动 无其他异常现象 即可退出旧nginx master进程
[nginx@lkarrie sbin]$ kill -QUIT 69168
# 注意 
# 如果旧worker进程没有完全停止 kill -QUIT 无法关闭旧master进程
# 需要等待所有旧worker进程退出后 再执行kill -QUIT 退出旧master进程


# 在没有退出旧nginx master进程之前 如果有不可控因素导致需要故障回滚
# 下面演示回滚步骤

# 旧nginx二进制文件 覆盖问题nginx二进制执行文件
[nginx@lkarrie sbin]$ mv nginx.old nginx

# 确认旧nginx执行文件替换成功
[nginx@lkarrie sbin]$ ./nginx -V
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --add-module=/app/nginx_build/module/nginx-module-vts-0.2.1 --add-module=/app/nginx_build/module/nginx_upstream_check_module

# 旧nginx master进程存在
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77181  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77182  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77183  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   78267  53248  0 05:28 pts/1    00:00:00 grep --color=auto nginx

# 唤醒旧nginx master进程 执行旧nginx二进制执行文件
[nginx@lkarrie sbin]$ kill -HUP 69168

# 旧nginx worker参与nginx请求处理
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77181  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77182  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   77183  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   78336  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78337  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78338  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78339  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78345  53248  0 05:29 pts/1    00:00:00 grep --color=auto nginx

# 优雅关闭问题nginx主进程和工作进程
[nginx@lkarrie sbin]$ kill -QUIT 77179

# 升级故障回滚完毕
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   78336  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78337  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78338  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78339  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78562  53248  0 05:32 pts/1    00:00:00 grep --color=auto nginx

# 验证
[nginx@lkarrie sbin]$ ./nginx -V
nginx version: nginx/1.20.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/app/nginx --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --add-module=/app/nginx_build/module/nginx-module-vts-0.2.1 --add-module=/app/nginx_build/module/nginx_upstream_check_module
```

**注**：上述演示的是升级操作，执行热重启不需要更换二进制文件，直接发送USR2 WINCH QUIT信号即可



### 日志管理

生产实践中，NGINX日志需要固定目录，供filebeat（或其他日志全家桶）抓取灌入可视化页面（例如kibana）中展示

而且NGINX日志格式需要有一些要求，方便排查线上问题

个人比较喜欢的日志格式和目录

* access日志（七层）：/app/logs/nginx/access-$server_name-$server_port-$logdate.log

* access日志（四层）：/app/logs/nginx/access-$protocol-$server_port-$logdate.log

* error日志：/app/logs/nginx/error.log

具体配置如下

**注**：日志配置实现了按天自动切割 access.log

```nginx
http{
    
    #....
    
    map $time_iso8601 $logdate {
        '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
        default    'date-not-found';
    }
    log_format  access  '[$time_iso8601] $remote_user $remote_addr:$remote_port "$server_protocol $request_method $scheme://$http_host$request_uri" '
                        '$status $body_bytes_sent $request_time $request_length "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for" '
                        '"$upstream_addr" $upstream_status $upstream_response_time';
                    
    access_log /app/logs/nginx/access-$server_name-$server_port-$logdate.log access;
    
    #....
}

stream{

    #....
    map $time_iso8601 $logdate {
      '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
      default    'date-not-found';
    }

    log_format stream '[$time_iso8601] $remote_addr:$remote_port $protocol $server_addr:$server_port '
                    '$status $bytes_sent $bytes_received $session_time '
                    '"$upstream_addr" "$upstream_bytes_received" "$upstream_bytes_sent" "$upstream_connect_time"';

    access_log /app/logs/nginx/access-$protocol-$server_port-$logdate.log stream;
    #....
}
```



当然NGINX的日志切割也可以使用logrotate，下面是具体方法（个人觉得直接使用配置实现日志切割较为方便

使用logrotate工具切割，crontab定时每天0点切割access.log和error.log，保存每天0-24时nginx日志

执行命令 vim /app/logs/nginx/nginx-logrotate-conf 创建如下配置文件

```shell
/app/logs/nginx/*.log
{  
    daily 
    rotate 30
    dateext
    create 0644 nginx nginx 
    compress
    delaycompress
    notifempty
    sharedscripts 
    postrotate
      ProcNumber=$(ps -ef |grep -w nginx|wc -l)
      if [ ${ProcNumber} -gt 1 ];then  
        nginx -s reopen
      fi
    endscript
}
```

执行命令 crontab -e 创建定时任务

```shell
0 0 * * * /usr/sbin/logrotate -f /app/logs/nginx/nginx-logrotate-conf >/dev/null 2>&1
```



### 开机自启

利用crontab

**注**：注意添加 -c -p 参数，如果需要覆盖编译参数的话

```shell
@reboot /app/nginx/sbin/nginx
```



### 监控

社区版本的NGINX监控手段其实不多，开源可用的module [ngx_http_stub_status_module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) 展示的内容也太少了

如果你用的是开源社区版本的NGINX，非常推荐使用一下三方模块 [vts](https://github.com/vozlt/nginx-module-vts) 这个真是一个良心的模块 

NGINX 构建加入这个模块之后 添加一些vts配置 即可以通过 http的http页面（页面访问路径是可以配置 http://ip:port/location）或Prometheus指标 查看NGINX的很多信息

```nginx
http{
    
    # ....
    # 设置 vts共享内存
    vhost_traffic_status_zone;
    # 开启自定义filter分组监控
    vhost_traffic_status_filter on;
    # 按请求状态和配置server_name 分组
    vhost_traffic_status_filter_by_set_key $status $server_name;

    # 访问 vts 监控的 server 配置
    server {
         # 访问 vts 监控的端口
         listen 9913;
         # 访问 vts 监控的路径
         location /status {
           access_log off;
           # 允许自身访问 例如 curl测试
           allow 127.0.0.1;
           # allow prometheus ip
           # allow 1.1.1.1;    
           # 禁止除了白名单ip以外的所有访问 这个很重要 
           deny all;
           # 默认展示 html 页面显示监控状态
           vhost_traffic_status_display;
           vhost_traffic_status_display_format html;
         } 
    } 
    
    # ....
}
```



引用一下官方的一张图，展示一下vts的监控html页面

<img src="assets/nginx-vts" alt="nginx-vts" style="zoom: 80%;" />

详细分析一下这个html展示的内容

* Server Main 

  server main 主要展示了 当前NGINX的总体状态，例如运行机器的 hostname、nginx版本、nginx进程最后一次更新或启动到现在的时间、总体的链接情况、总体的请求情况、和vts所依赖的共享内存的区域状态（vts模块记录的这些指标存储在这里）

  <img src="assets/nginx-vts-servermain" alt="nginx-vts-servermain" style="zoom:80%;" />

  

* Server Zone

  server zone 主要是对每一个 server 配置下面的 请求处理的状态，你可以查看你的每一个nginx server 配置下的请求状态（例如请求响应状态1xx 2xx 3xx 4xx 5xx的情况、流量情况等）

  <img src="assets/nginx-vts-serverzones" alt="nginx-vts-serverzones" style="zoom:80%;" />
  
  **注意**：没有设置server_name的server vts会以 _ 代替名称，并自动生成具体的请求状态码，例如上图的 200，400，都是指没有设置server_name的server请求响应情况
  
  

* Filters

  filters 在官方的样例图中没有具体展示，但是它很重要，主要是通过filter来实现自定监控项

  官方配置中和filters相关的配置如下

  ```nginx
  http{
      #...
      # 开启自定义filter分组监控
      vhost_traffic_status_filter on;
      # 按请求状态和配置server_name 分组
      # 下面vts配置 是设置监控 每个nginx server下的具体请求状态码
      vhost_traffic_status_filter_by_set_key $status $server_name;
      #...
      # 下面配置一些自己的server和vts无关
      server {
          #...
          server_name *.lkarrie.com;
          #...
  	}
      server {
          #...
          server_name blog.lkarrie.com;
          #...
      }
      #...
  }
  ```

  实现效果如下，可以看到filters对应的group为server下的server_name，每个gourp监控的key为具体的httpcode 200、206、301等等

  <img src="assets/nginx-vts-filters" alt="image-20230607085504775" style="zoom:80%;" />

  **vhost_traffic_status_filter_by_set_key 后关于key的设置除了 $status 你也可以设置其他变量，如果设置了其他变量就相当于监控每个server下的这个变量维度的1xx 2xx 3xx 4xx 5xx 出入流量 等状态**

  举个监控server下具体接口状态的例子，增加一个自定义变量拦截特定的接口

  按照如下的配置，最终filters下的Zone为自定义变量$alerturl

  当然你可以根据实际的需求将map下的正则改成别的，实现特定规则的接口监控

  ```nginx
  http{
      #...
      # 当请求路径包含gateway时 为$alerturl设置为当前请求的uri
      # 不包含gateway的请求$alerturl统一设置为/not-alarm-request
      map $uri $alerturl {
          ~*(gateway) $uri;
          default '/not-alarm-request';
      }
      
      # 自定义 vts共享内存
      # 由于我们监控的接口可能一直增长或数量较多 需要适当调整vts共享内存的默认大小
      # 满了虽然不会影响使用，但是会停止记录新的key
      vhost_traffic_status_zone shared:vhost_traffic_status:300m;
      # 开启自定义filter分组监控
      vhost_traffic_status_filter on;
      # 按请求状态和配置server_name 分组
      vhost_traffic_status_filter_by_set_key $alerturl $server_name;
      #...
  }
  ```

  

* Upstreams

  

* Caches



### 高可用



## 常用技巧和配置



