# NGINX Notes



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

注意：openssl 版本 和TLS 版本有关

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

./configure --prefix=/app/sunline/nginx-1.18.0 --with-compat --with-file-aio --with-threads --with-http_ssl_module --with-stream --with-stream_ssl_module --with-http_sub_module --add-module=/app/nginx_build/module/nginx-module-vts

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



