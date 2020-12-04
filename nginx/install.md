## nginx 编译
```base
./configure --prefix=/usr/local/nginx \
--user=nginx \
--group=nginx \
--with-http_v2_module \
--with-http_ssl_module \
--with-http_realip_module \
--pid-path=/var/run/nginx.pid \
--error-log-path=/var/log/error.log \
--http-log-path=/var/log/access.log
```



## 配置

- nginx.conf

> nginx作为http服务器的时候：    max_clients = worker_processes * worker_connections  
>
> nginx作为反向代理服务器的时候：    max_clients = worker_processes * worker_connections/4

```
user  nginx nginx;
worker_processes  auto;
worker_rlimit_nofile 65535;

error_log  /var/log/nginx/error.log  crit;

pid        /var/run/nginx.pid;


events {
    worker_connections  65535;
    multi_accept on;
}


http {
    include       real_ip;
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  /var/log/nginx/access.log  main;

    client_max_body_size 20m;
    client_body_buffer_size 10m;

    sendfile        on;
    #tcp_nopush     on;
    #tcp_nodelay    on;

    keepalive_timeout  120;

    gzip  on;
    gzip_buffers  32 4k;
    gzip_comp_level  6;
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private auth;
    gzip_vary on;
    gzip_types text/plain text/javascript text/css application/javascript application/xml 
        image/jpeg image/gif image/png font/ttf font/opentype font/x-woff;

    fastcgi_connect_timeout 60s;
    fastcgi_send_timeout 120s;
    fastcgi_read_timeout 60s;

    include conf.d/*.conf;
}
```

- conf.d/default.conf

> nfs 作为 nginx 的 `root` 时，性能和本地大概会降低10倍左右

```
server {
    listen 80;
    listen 443 ssl http2;
    server_name domain.com;

    ssl_certificate     /cert/domain.com.crt;
    ssl_certificate_key /cert/domain.com.key; 

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 10m;
    ssl_session_cache builtin:1000 shared:SSL:10m;
    ssl_buffer_size 1400;

    access_log  /var/log/nginx/access.log  main;
    error_log   /var/log/nginx/error.log;

    root /usr/local/nginx/html;
    index index.html index.php;

    location ~ \.php {

    }
    
}
```



## 日志切割

`/etc/logrotate.d/nginx`

```
/var/log/nginx/*.log{
    weekly
    rotate 12
    missingok
    dateext
    compress
    notifempty
    sharedscripts
    postrotate
        [ -e /var/run/nginx.pid ] && /bin/kill -USR1 $(cat /var/run/nginx.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

强制轮询切割日志:  `logrotate -vf /etc/logrotate.d/nginx` (测试)