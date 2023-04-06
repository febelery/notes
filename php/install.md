## 依赖

### ubuntu

```
sudo apt install -y libsqlite3-dev libssl-dev libcurl4-openssl-dev libjpeg-dev libonig-dev libzip-dev autoconf libicu-dev libfreetype6-dev libsodium-dev unzip
```

### centos 8

```
yum install libjpeg-turbo-devel libpng-devel libffi-devel libcurl-devel sqlite-devel libxml2-devel openssl-devel pcre-devel libzip-devel freetype-devel libicu-devel gcc-c++ libsodium-devel
```

#### oniguruma

 oniguruma是一个处理正则表达式的库，mbstring的正则表达式处理功能对这个包有依赖性

```bash
yum install autoconf automake libtool
wget https://github.com/kkos/oniguruma/archive/v6.9.6.tar.gz -O oniguruma-6.9.6.tar.gz 
tar -zxvf oniguruma-6.9.6.tar.gz  && cd oniguruma-6.9.4/ 
./autogen.sh && ./configure --prefix=/usr
make && make install
```



## 源码编译

- `--with-pear`这个参数是用于生成`pecl`命令

```bash
./configure  \
	--prefix=/usr/local/php \
	--enable-opcache \
	--enable-opcache-jit \
	--enable-intl \
	--enable-fpm \
	--enable-bcmath \
	--enable-mbstring \
	--enable-pdo \
	--enable-dom \
	--enable-simplexml \
	--enable-session \
	--enable-sockets \
	--enable-calendar \
	--enable-pcntl \
	--enable-shmop \
	--enable-ipv6 \
	--with-iconv \
	--with-bz2 \
	--with-pdo-mysql \
	--with-mysqli \
	--enable-mysqlnd \
	--enable-fileinfo \
	--enable-mbregex \
	--enable-gd \
	--with-jpeg \
	--with-freetype \
	--with-curl \
	--with-zip \
	--with-openssl \
	--with-pcre-jit \
	--with-ffi \
	--with-mhash \
	--with-zlib \
	--with-zlib-dir \
	--with-pear \
	--with-gettext \
	--with-sodium
```



## 配置

### php-fpm

```
[global]
pid = /var/run/php-fpm.pid
error_log = /var/log/php/php-fpm.log
```

### backlog

backlog是linux下socket函数之listen的参数，当应用程序调用listen系统调用让一个socket进入LISTEN状态时，需要指定一个backlog参数。这个参数经常被描述为，新连接队列的长度限制。

由于TCP建立连接需要进行3次握手，一个新连接在到达ESTABLISHED状态可以被accept系统调用返回给应用程序前，必须经过一个中间状态SYN RECEIVED。

**已连接但未进行accept处理的SOCKET队列大小**，如果这个队列满了，将会发送一个ECONNREFUSED错误信息给到客户端。用两个backlog来分别限制半连接SYN_RCVD状态的未完成连接队列大小跟全连接ESTABLISHED状态的已完成连接队列大小

> php-fpm的默认backlog是511
>
> backlog值为65535太大了，会导致前面的nginx(或者其他客户端)超时
>
> 假设FPM的QPS为5000，那么65535个请求全部处理完需要13s的样子。但前端的nginx(或其他客户端)已经等待超时，关闭了这个连接。当FPM处理完之后，再往这个SOCKET ID 写数据时，却发现连接已关闭，得到的是“error: Broken Pipe”，在nginx、redis、apache里，默认的backlog值都是511

backlog太大了，导致FPM处理不过来，nginx那边等待超时，断开连接，报504 gateway timeout错。同时FPM处理完准备write 数据给nginx时，发现TCP连接断开了，报“Broken pipe”。

backlog太小的话，NGINX之类client，根本进入不了FPM的accept queue，报“502 Bad Gateway”错。所以，这还得去根据FPM的QPS来决定backlog的大小。

计算方式最好为QPS=backlog



ulimit -a

```
net.core.somaxconn = 512
net.core.netdev_max_backlog = 1000
net.ipv4.tcp_max_syn_backlog = 2048
```

SYN queue长度由tcp_max_syn_backlog指定，accept queue则由net.core.somaxconn决定，listen(fd, backlog)的backlog上限由somaxconn决定

![backlog](https://upload-images.jianshu.io/upload_images/3407216-a681460806d3b6de.png)

SYN队列(待完成连接队列)和accept队列(已完成连接队列)。状态为SYN RECEIVED的连接进入SYN队列，后续当状态变更为ESTABLISHED时移到accept队列(即收到3次握手中最后一个ACK包)。顾名思义，accept系统调用就只是简单地从accept队列消费新连接。在这种情况下，listen系统调用backlog参数决定accept队列的最大规模



### pm

当`pm = dynamic`时，可根据每个php-fpm占用的内存大小设置`pm.max_children`，

`真实占用内存 = 常驻内存 - 共享内存`，经过测试，每个php-fpm占用大概在3M到几十M之间

> 比如一个8G内存的服务器，设置`pm = static` / `pm.max_children = 1600`，内存从5200M降为400M左右

>**VIRT：virtual memory usage 虚拟内存
>**1、进程“需要的”虚拟内存大小，包括进程使用的库、代码、数据等
>2、假如进程申请100m的内存，但实际只使用了10m，那么它会增长100m，而不是实际的使用量
>
>**RES：resident memory usage 常驻内存**
>1、进程当前使用的内存大小，但不包括swap out
>2、包含其他进程的共享
>3、如果申请100m的内存，实际使用10m，它只增长10m，与VIRT相反
>4、关于库占用内存的情况，它只统计加载的库文件所占内存大小
>
>**SHR：shared memory 共享内存**
>1、除了自身进程的共享内存，也包括其他进程的共享内存
>2、虽然进程只使用了几个共享库的函数，但它包含了整个共享库的大小
>3、计算某个进程所占的物理内存大小公式：RES – SHR
>4、swap out后，它将会降下来
>
>**DATA**
>1、数据占用的内存。如果top没有显示，按f键可以显示出来。
>2、真正的该程序要求的数据空间，是真正在运行中要使用的。

![](https://img.orchome.com/group1/M00/00/00/KmCudld5_zmAajUHAABd80xLId0540.png)





### systemd

`/usr/lib/systemd/system/php-fpm.service`

```
[Unit]
Description=The PHP FastCGI Process Manager
After=network.target

[Service] 
Type=simple
PIDFile=/var/run/php-fpm.pid
ExecStart=/usr/local/php/sbin/php-fpm
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -INT $MAINPID

[Install]
WantedBy=multi-user.target
```

