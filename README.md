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

> 记录一些常用的安装步骤



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
[root@nginx nginx_build]# tree -L 1
.
├── module
├── module_tar
├── nginx-1.18.0
├── nginx-1.18.0.tar.gz
├── nginx-1.20.2
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
│   └── conf.d
├── fastcgi_temp
├── html
├── logs
│   └── nginx.pid
├── proxy_temp
├── sbin
├── scgi_temp
├── script
└── uwsgi_temp
```



## 运维

> 记录一些常用的运维操作



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

# 没有执行新二进制文件 所以进程pid无变化
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   76801  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   76959  53248  0 05:13 pts/1    00:00:00 grep --color=auto nginx

# 执行升级nginx二进制文件 USR2信号
[nginx@lkarrie sbin]$ kill -USR2 69168

# 启动新的nginx master worker进程
# 此时新旧nginx worker进程将同时处理请求
# 如果此时没有新的 master worker进程启动说明nginx conf配置存在异常 无法启动nginx
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   76801  69168  0 05:11 ?        00:00:00 nginx: worker process
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
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
nginx   78267  53248  0 05:28 pts/1    00:00:00 grep --color=auto nginx

# 唤醒旧nginx master进程 执行旧nginx二进制执行文件
[nginx@lkarrie sbin]$ kill -HUP 69168

# 旧nginx worker参与nginx请求处理
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77179  69168  0 05:16 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   77180  77179  0 05:16 ?        00:00:00 nginx: worker process
nginx   78336  69168  0 05:29 ?        00:00:00 nginx: worker process
nginx   78345  53248  0 05:29 pts/1    00:00:00 grep --color=auto nginx

# 优雅关闭问题nginx主进程和工作进程
[nginx@lkarrie sbin]$ kill -QUIT 77179

# 升级故障回滚完毕
[nginx@lkarrie sbin]$ ps -ef | grep nginx
nginx   69168      1  0 04:26 ?        00:00:00 nginx: master process ../sbin/nginx
nginx   78336  69168  0 05:29 ?        00:00:00 nginx: worker process
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



### 日志分析

日志分析相关，还是最好收拢到ES里，方便检索

下面补充一些小工具分析

goaccess分析日志 [整好之后的效果]([服务器统计信息 (lkarrie.com)](http://goaccess.lkarrie.com/goaccess/))

```markdown
# 离线ip库
yum install -y GeoIP-devel --downloadonly --downloaddir=./
yum install GeoIP-1.5.0-14.el7.x86_64.rpm GeoIP-devel-1.5.0-14.el7.x86_64.rpm geoipupdate-2.5.0-1.el7.x86_64.rpm -y

# 安装goaccess
wget https://tar.goaccess.io/goaccess-1.7.2.tar.gz
tar -xzvf goaccess-1.7.2.tar.gz
cd goaccess-1.7.2
./configure --enable-utf8 --enable-geoip=legacy --with-getline --with-openssl
make
make install

# 配置
vim /usr/local/etc/goaccess/goaccess.conf
# 增加配置
time-format %T
date-format %Y-%m-%d
log-format [%dT%t+%^] %^ %h:%^ "%H %m %^ %v %U" %s %b %T %^ "%R" "%u" "~h{,}"  %^ %^ %^

# 注意这里需要根据自己的 nginx logformat进行配置
# 参考官方文档 https://goaccess.io/man#custom-log
# 上面配置对应的 log_format
log_format  access  '[$time_iso8601] $remote_user $remote_addr:$remote_port "$server_protocol $request_method $scheme $http_host $uri" '
'$status $body_bytes_sent $request_time $request_length "$http_referer" '
'"$http_user_agent" "$http_x_forwarded_for" '
'"$upstream_addr" $upstream_status $upstream_response_time';
##### 

# 跑一下生成html
# 可以写脚本放到后台跑
# nginx加一下配置就能实时访问这个资源了
# 注意开放 websocket 7890端口
goaccess /app/logs/nginx/access.log -o /app/nginx/html/goaccess/index.html --real-time-html
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

<img src="assets/nginx-vts" alt="nginx-vts" style="zoom: 50%;" />

详细分析一下这个html展示的内容

* Server Main 

  server main 主要展示了 当前NGINX的总体状态，例如运行机器的 hostname、nginx版本、nginx进程最后一次更新或启动到现在的时间、总体的链接情况、总体的请求情况、和vts所依赖的共享内存的区域状态（vts模块记录的这些指标存储在这里）

  <img src="assets/nginx-vts-servermain" alt="nginx-vts-servermain" style="zoom: 50%;" />

  

* Server Zone

  server zone 主要是对每一个 server 配置下面的 请求处理的状态，你可以查看你的每一个nginx server 配置下的请求状态（例如请求响应状态1xx 2xx 3xx 4xx 5xx的情况、流量情况等）

  <img src="assets/nginx-vts-serverzones" alt="nginx-vts-serverzones" style="zoom: 50%;" />
  
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

  <img src="assets/nginx-vts-filters" alt="image-20230607085504775" style="zoom: 50%;" />

  **vhost_traffic_status_filter_by_set_key 后关于key的设置除了 $status 你也可以设置其他变量，如果设置了其他变量就相当于监控每个server下的这个变量维度的1xx 2xx 3xx 4xx 5xx 出入流量 等状态**

  举个监控server下具体接口状态的例子，增加一个自定义变量捕获特定的接口

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

  upstream主要包含了upstream包含的server信息，例如当前server被动探测的参数 up/down状态 以及1xx 2xx 3xx 4xx 5xx 和 出入流量的信息

  ```nginx
  http{
      #...
      upstream group1 {
          # 按upstream的名称（group1）展示upstream下的server信息
          server 127.0.0.1:8080;
          server 127.0.0.1:8081;
      }
      #...
  }
  ```

  可以通过这里的信息判断上游的节点是否存活、或者根据响应时间判断上游是否存在异常

  **注意**：需要额外说明一下 ::nogroups 下server展示的是未用upstream声明的上游地址（在 proxy_pass 后置设置的地址）

  ```nginx
  http{
      #...
      server{
          #...
          location / {
              # proxy_pass 后直接配置实际地址的状态 vts将在::nogroups下展示
              proxy_pass http://127.0.0.1:8082;
          }
          #...
      }
      #...
  }
  ```

  

* Caches

  关于Caches个人暂时还没有用到，vts上没有相关展示，应该和nginx缓存配置相关，后续使用到了再补充




当然除了使用html方式你通过http的形式获取vts的Prometheus指标

```terminal
# 通过format/prometheus 请求vts配置返回Prometheus指标
[root@lkarrie ~]# curl http://127.0.0.1:9913/status/format/prometheus
# HELP nginx_vts_info Nginx info
# TYPE nginx_vts_info gauge
nginx_vts_info{hostname="lkarrie",module_version="v0.2.1",version="1.20.2"} 1
# HELP nginx_vts_start_time_seconds Nginx start time
# TYPE nginx_vts_start_time_seconds gauge
nginx_vts_start_time_seconds 1686053290.672
```

既然它支持输出Prometheus指标，当然就可以展示到Grafana中或对接Prometheus Alertmanager 根据Nginx Vts指标进行告警，实际生产中我们也是正是这么做的

有一些改动的Nginx Vts Grafana监控大盘

<img src="assets/nginx-vts-grafana" alt="nginx-vts-grafana" style="zoom: 50%;" />



### 高可用

* 在传统虚机部署中，你可以依赖keepalived做软VIP的主备架构

![nginx-unit2](assets/nginx-unit2)

​	（有时间补充一下keepalived - 20230607）



* 或者依赖硬件设备，做负载均衡形成集群

![nginx-unit1](assets/nginx-unit1)



* 再者就是上云依赖平台底座例如K8S等，提供容错、自愈、和横向扩展的功能



## 常用配置

> 记录一些个人使用或生产实践过程中常用的NGINX配置



### 长链接

#### HTTP 长链接

在NGINX 七层代理中使用长链接（复用TCP通道）

实际生产中 keepalive 设置的值需要慎重，而且这个值是单独制定一个worker进程，实际的keepalive数需要和worker进程相乘，推荐根据实际业务的tps推算到每个worker进程的keepalive链接数并做微上调，保证需求避免keepalive设置值不合理

```nginx
http{
    #...
    upstream keepalive {
        server 192.168.0.4:8080;
        server 192.168.0.5:8080;
        # 最大空闲的keepalive链接数
        # 需要注意这个配置并不是限制 keepalive的链接数
        keepalive 32;
        # 单个keepalive链接最大处理的请求数
        keepalive_requests 10000;
        # 空闲最大时间
        keepalive_timeout 60s;
        # 最大存活时间 1.19.10 版本后可配置
        keepalive_time 1h;
    }
    
    server {
        listen 8000;
        location / {
            proxy_set_header Connection "";
            proxy_http_version 1.1;
            proxy_pass http://keepalive;
        }
    }
    #...
}
```



#### STREAM 长链接

在NGINX的四层代理中使用长链接

```nginx
stream {
    #...
    proxy_connect_timeout 5s;
    proxy_socket_keepalive on;
    
    upstream test {
        server 192.168.0.3:2000;
    }
    
    server {
        listen 2001;
        proxy_pass test;
    }
    #...
}
```



### 反向代理

> 反向代理主要指客户端访问代理服务器（NGINX）之后，反向代理服务器（NGINX）根据一定的规则（location、proxy_pass）从被一个或多个被代理服务器中获取响应资源并返回给客户端的模式

反向代理有两种

* http的反向代理

  http的反向代理在日常工作中用的最多，在局域网中可以利用nginx反向代理将应用的外发流量转到对应的合作方机构

* https的反向代理

  https的反向代理通常放在dmz区域，做静态资源的代理，这种也叫做NGINX的SSL卸载或者终止



#### HTTP反向代理

HTTP的反向代理并不困难，不论对内或对外都是配置proxy_pass即可，但是实际生产使用中，对外的反向代理其实是很复杂的，通常对方只会提供给我们一个域名，反向代理域名的情况下，就会涉及域名的动态解析、实际IP的缓存、甚至NGINX与DNS服务器的交互的问题等等

下面的一些配置仅针对**NGINX社区版本**，商业版本的NGINX Plus对上面的一些实际问题都有很好的解决方案（并不是所有老板都有钱支持采购商业版本~

在默认的情况下，如果你的配置文件里 proxy_pass 存在域名，而且并没有配置resolver，nginx只会在首次启动时（进程完全退出后启动）将域名对应的实际地址缓存到内存中，这个时候如果域名对应的实际地址变化之后，nginx其实是探测不到的

**在官方文档中，上述的情况是可以通过配置 resolver的 valid 参数来设置域名对应实际IP的时间，但在我工作当中用到的 1.18.0 1.20.2 NGINX这个参数并不能生效**...

最后我们通过变量设置域名的方式，保证每次请求都是去访问DNS服务器获取域名对应的IP，下面是一些参考配置

```nginx
http {
    #...
    server {
        #...
        set $baidu www.baidu.com
        resolver 192.168.0.2 ipv6=off valid=30s;
        resolver_timeout 3s;
        
        location / {
            proxy_pass https://$baidu:8443;
            proxy_set_header Host $baidu;
        }
        #...
    }
    #...
}
```

**注意**：上述的配置其实也是有弊端的，变量是无法在upstream中使用的，如果不使用upstream，nginx http的长链接就无法使用了，只能全部通过短连接方式，如果想使用长链接，并且upstream server中配置了域名，这又无法做到域名对应实际ip地址的动态更新了



#### HTTPS反向代理

HTTPS我使用的不是很多，目前工作中SSL卸载是在硬件上实现的，证书同样也是在硬件上

后续生产实践中使用积累经验或遇到事故问题再补充补充

推荐一个SSL 配置生成网站，可以一键生成NGINX的SSL配置

[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)

下面是个人博客使用到的一些配置，仅供参考

```nginx
http {
    #...
    server {
        listen 443 ssl;
        server_name lkarrie.com;
        ssl_certificate  cert/root/lkarrie.com.pem;
        ssl_certificate_key cert/root/lkarrie.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;        
        ssl_prefer_server_ciphers off;

        rewrite ^(.*) https://blog.lkarrie.com$1 permanent;
	}
    #...
}
```

**注意**：如果你的网站配置了 ssl_protocols TLSv1.3，但是检测网站TLS支持版本并不包含TLS 1.3，大概率是NGINX编译的openssl版本并不支持TLS 1.3



### 正向代理

### 静态缓存

### 逻辑判断

### 安全配置



## 常见的状态码问题分析

### 101

### 400

### 408

### 413

### 499

### 502

### 504

