# Linux常用命令

## lsof

展示一个进程打开的所有文件，或者打开一个文件的所有进程

```bash
$ ps -ef | grep php | grep -v grep | head -n 1 | awk '{print $2}' | xargs lsof -p

COMMAND    PID USER   FD      TYPE             DEVICE SIZE/OFF     NODE NAME
php-fpm 609360 root  cwd       DIR              253,1      256      128 /
php-fpm 609360 root  rtd       DIR              253,1      256      128 /
php-fpm 609360 root  txt       REG              253,1 60812112 18156621 /usr/local/php/sbin/php-fpm
php-fpm 609360 root  DEL       REG                0,5          30710519 /dev/zero
php-fpm 609360 root  DEL       REG                0,5          30711231 /dev/zero
php-fpm 609360 root  mem       REG              253,1  6406312 51348518 /var/lib/sss/mc/group
php-fpm 609360 root  mem       REG              253,1  8406312 51348514 /var/lib/sss/mc/passwd
php-fpm 609360 root  mem       REG              253,1    38160 51159590 /usr/lib64/libnss_sss.so.2
php-fpm 609360 root  mem       REG              253,1  3664328 84002840 /usr/local/php/lib/php/extensions/no-debug-non-zts-20200930/redis.so
```

## iostat

监视系统输入输出设备和CPU的使用情况。它的特点是汇报磁盘活动统计情况，同时也会汇报出CPU使用情况

iowait并不能反应磁盘瓶颈，iowait实际测量的是cpu时间，%iowait = (cpu idle time)/(all cpu time)

高速cpu会造成很高的iowait值，但这并不代表磁盘是系统的瓶颈。唯一能说明磁盘是系统瓶颈的方法，就是很高的read/write时间，一般来说超过20ms，就代表了不太正常的磁盘性能。为什么是20ms呢？一般来说，一次读写就是一次寻到+一次旋转延迟+数据传输的时间。由于，现代硬盘数据传输就是几微秒或者几十微秒的事情，远远小于寻道时间2~20ms和旋转延迟4~8ms，所以只计算这两个时间就差不多了，也就是15~20ms。只要大于20ms，就必须考虑是否交给磁盘读写的次数太多，导致磁盘性能降低了

```bash
$ iostat -x 1

Linux 4.18.0-193.28.1.el8_2.x86_64 (votetemp001)        2020年12月09日  _x86_64_        (4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.88    0.00    0.74    0.12    0.00   98.25

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
vda              1.85    0.45    256.94     16.09     0.02     0.14   0.82  24.45   60.21    7.09   0.11   138.77    36.11   0.36   0.08
vdb              0.00    0.00      0.01      0.00     0.00     0.00   0.00   0.00    1.56    0.00   0.00    21.99     0.00   0.73   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.50    0.00    0.50    0.00    0.00   99.00

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
vda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00
vdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00
```

- await:   平均每次设备I/O操作的等待时间 (毫秒)。即 delta(ruse+wuse)/delta(rio+wio)
- svctm:  平均每次设备I/O操作的服务时间 (毫秒)。即 delta(use)/delta(rio+wio)
- %util:    一秒中有百分之多少的时间用于 I/O 操作，或者说一秒中有多少时间 I/O 队列是非空的。即 delta(use)/s/1000 (因为use的单位为毫秒)

如果 %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。idle小于70% IO压力就较大了,一般读取速度有较多的wait。

svctm 一般要小于 await (因为同时等待的请求的等待时间被重复计算了)，svctm 的大小一般和磁盘性能有关，CPU/内存的负荷也会对其有影响，请求过多也会间接导致 svctm 的增加。await 的大小一般取决于服务时间(svctm) 以及 I/O 队列的长度和 I/O 请求的发出模式。如果 svctm 比较接近 await，说明 I/O 几乎没有等待时间；如果 await 远大于 svctm，说明 I/O 队列太长，应用得到的响应时间变慢，如果响应时间超过了用户可以容许的范围，这时可以考虑更换更快的磁盘，调整内核 elevator 算法，优化应用，或者升级 CPU



