

# 性能测试

服务器配置(aliyun)，其他参考 [php install](./install.md) 和 [nginx install](../nginx/install.md)

- `centos 8`
- 4核8G内存
- nginx/1.17.10
- php/8.0.0
  - pm = dynamic
  - pm.max_children = 256
  - pm.start_servers = 2
  - pm.min_spare_servers = 2
  - pm.max_spare_servers = 4 
- 挂载nfs当做代码盘
  - mount -t nfs -o vers=3,proto=tcp,noresvport xxxx.cn-hangzhou.extreme.nas.aliyuncs.com:/ /data

> nfs与local比较
>
> nginx的性能大概有10倍的下降，local qps **136694**，nfs qps **12600**
>
> 对PHP影响不大，估计是php的性能还跟不上nginx的压测极限，以下数据全部基于nfs目录测试



## php

显示`phpinfo`页面

#### 开启opcache

```
load average: 19.14, 8.05, 5.04

Running 1m test @ http://127.0.0.1/test.php
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   194.83ms  327.44ms   1.93s    85.91%
    Req/Sec     0.85k   242.52     2.06k    68.24%
  Latency Distribution
     50%   41.85ms
     75%   54.04ms
     90%  767.42ms
     99%    1.26s 
  203244 requests in 1.00m, 15.00GB read
  Socket errors: connect 0, read 0, write 0, timeout 1058
  Non-2xx or 3xx responses: 120
Requests/sec:   3381.67
Transfer/sec:    255.49MB
```



#### 关闭opcache

```
load average: 23.27, 9.32, 6.14

Running 1m test @ http://127.0.0.1/test.php
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   166.53ms  284.76ms   1.97s    88.63%
    Req/Sec   748.36    240.34     1.54k    68.01%
  Latency Distribution
     50%   52.33ms
     75%   81.35ms
     90%  710.05ms
     99%    1.15s 
  179115 requests in 1.00m, 13.04GB read
  Socket errors: connect 0, read 0, write 0, timeout 1019
  Non-2xx or 3xx responses: 114
Requests/sec:   2981.47
Transfer/sec:    222.31MB
```





## laravel

```shell
composer install --optimize-autoloader --no-dev
php artisan config:cache
php artisan route:cache
php artisan view:cache
wrk -t4 -c1000 -d60s --latency http://127.0.0.1
```

`session`和`cache`用**redis**

### view模板

#### 开启opcache

```
load average: 24.84, 14.03, 6.65

Running 1m test @ http://127.0.0.1
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   391.80ms  396.07ms   2.00s    86.29%
    Req/Sec   167.18     86.85   580.00     61.84%
  Latency Distribution
     50%  241.58ms
     75%  257.96ms
     90%    1.25s 
     99%    1.69s 
  39732 requests in 1.00m, 698.26MB read
  Socket errors: connect 0, read 0, write 0, timeout 1518
  Non-2xx or 3xx responses: 404
Requests/sec:    661.07
Transfer/sec:     11.62MB
```



#### 关闭opcache

```
load average: 54.14, 14.46, 5.63

Running 1m test @ http://127.0.0.1
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.07s   544.26ms   2.00s    59.57%
    Req/Sec    14.04     25.87   340.00     97.20%
  Latency Distribution
     50%    1.01s 
     75%    1.51s 
     90%    1.77s 
     99%    2.00s 
  2509 requests in 1.00m, 35.17MB read
  Socket errors: connect 0, read 0, write 0, timeout 2462
  Non-2xx or 3xx responses: 537
Requests/sec:     41.76
Transfer/sec:    599.33KB
```



### 数据库

全部**开启opcache**

#### SQL执行时间小于20ms

```
load average: 18.64, 8.73, 6.04

Running 1m test @ http://127.0.0.1
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   424.05ms  371.20ms   2.00s    87.94%
    Req/Sec   142.45     59.00     0.94k    73.19%
  Latency Distribution
     50%  286.87ms
     75%  318.99ms
     90%    1.29s 
     99%    1.71s 
  34018 requests in 1.00m, 43.20MB read
  Socket errors: connect 0, read 0, write 0, timeout 1601
  Non-2xx or 3xx responses: 511
Requests/sec:    566.24
Transfer/sec:    736.33KB
```



#### SQL执行时间100ms-3s

```
load average: 2.25, 5.70, 5.31

Running 1m test @ http://127.0.0.1
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.90s    37.77ms   1.96s    57.14%
    Req/Sec    15.44     28.90   620.00     98.07%
  Latency Distribution
     50%    1.90s 
     75%    1.93s 
     90%    1.96s 
     99%    1.96s 
  2513 requests in 1.00m, 2.88MB read
  Socket errors: connect 0, read 0, write 0, timeout 2506
  Non-2xx or 3xx responses: 356
Requests/sec:     41.81
Transfer/sec:     49.04KB
```



## lumen



## yii2



## symfony



## thinkphp