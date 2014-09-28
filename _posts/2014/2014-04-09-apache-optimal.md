---
layout: post-it
title: "一次Apache性能优化"
date: 2014-04-09T23:30:00-08:00
categories:
  -- it
  -- arch
---
在keep-alive一节，尝试优化apache的线程数的时候，我一直带着两个疑问：一是为何我的进程占用内存高达100M之多；而是apache的线程数设置如何优化。还理解这一点，还得从apache
mpm认识起。

## 认识MPM

多路处理模块(MPM)是Apache2.x起支持的插入式并行处理模块，意在使Apache能在不同的平台和不同的环境实现最优效率。按官网的介绍，其带来两个重要的好处：

       1. Apache 能更优雅，更高效率的支持不同的平台。尤其是 Apache 的 Windows 版本现在更有效率了，因为 mpm_winnt
       能使用原生网络特性取代在 Apache 1.3 中使用的 POSIX 层。它也可以扩展到其它平台 来使用专用的 MPM。

       2. Apache 能更好的为有特殊要求的站点定制。例如，要求 更高伸缩性的站点可以选择使用线程的 MPM，即 worker 或 event；
       需要可靠性或者与旧软件兼容的站点可以使用 prefork。

常见的MPM：

prefork :
官网称其，是在"要求每个请求相互独立的情况"下表现最好的web服务器，其具有较强的自我调节能力，且仅需很少的配置指令就能得到较优的结果。关键设置MaxClients，这个值既要足够大到以处理潜在的请求高峰，同时又不能造成内存溢出(具体优化后面介绍)。

worker :
意在使网络服务器支持混合的多线程多进程，相比prefork的无线程，其利用了线程低开销的特点支持更大量的请求，不过其兼容性与可靠性弱于prefork，关键配置为ThreadsPerChild。值得注意的是，在worker模型下，子进程由为数不多的父进程建立，同时每个子进程可以建立ThreadsPerChild数量的线程和一个监听线程。

### 查看目前httpd的工作模式？

    apachectl -l

Compiled in modules:

    core.c
    prefork.c 
    http_core.c
    mod_so.c

表示apache当前的工作模式为prefork

配置工作模式：

apache默认支持prefork模型，需要支持worker mpm，需要在安装时指定

    ./configure --prefix=/usr/local/apache2worker --enable-so --with-mpm=worker
    make 
    make install

## 我们的Prefork模式

对于一般的企业级应用，基于稳定性考虑，大多选择这一模式。

缺省配置

       StartServers                  5

       MinSpareServers             5

       MaxSpareServers           10

       MaxClients                     150

       MaxRequestsPerChild        0

配置说明：

StartServers设定perfork启动时创建事务进程数量。

MinSpareServers，设定了最小的空闲进程数，也就是apache会一直保留这个数量的进程，哪怕它空闲了也不会被销毁。其创建的过程很有意思:perfork先创建一个进程，并在等待一秒后再创建两个进程，下一秒4个、再下一秒16个、最后32个(指数级)。并以后一直保持每秒创建32个进程的速度，直到满足MinSpareServers为止。

相对的，MaxSpareServers设定了最大的空闲进程数，防止资源的浪费。当MaxSpareServers
\<
MinSpareServers时，Apache会自动将其调整为MinSpareServers+1。值得注意的是，保持一定数量的MinSpareServers是很有必要的，因为其可以减少请求到来时apache
fork新进程的开销，但同时过多空闲进程数也会占用资源。

MaxRequestsPerChild意味着进程处理多少个请求后将自动销毁，0意味着无限，即永不销毁。负载较高时，为使每个进程处理更过的请求，避免销毁、创建线程的开销，一般建议设定这个值为0或较大的数字。但是要注意即使负载降低后，MaxRequestsPerChild也会造成进程占用的内存无法释放，造成"负载降低，但是系统开销居高不下的局面"。

MaxClients设定Apache可同时处理的请求数量，其对Apache性能的影响非常大。默认的150远远不能满足一般站点(ps
-ef|grep httpd|wc
-l)，超过这个数量的请求需要排队，直到前面的请求处理完毕。如果你的系统资源还剩很多，但是HTP访问却很缓慢，大多时候增加这个值将是问题得到缓解。理论上，这个值越多，其处理的请求也就越多，当然你的系统资源得支持，可是我不明白，为什么apache将这个值限定为256。幸好Apche也意识到这个问题，提供了一个ServerLimit指令来加大MaxClients。

## 线程数优化

Prefork优化的关键在于MaxClients与MaxRequestsPerChild。

**MaxClients**

最佳的值：

       apache_max_process_with_good_perfermance < (total_hardware_memory /
       apache_memory_per_process )  * 2 ；

       apache_max_process = apache_max_process_with_good_perfermance * 1.5

计算httpd平均占用内存

    ps aux|grep -v grep|awk '/httpd/{sum+=$6;n++};END{print sum/n}'

108468

(结果单位为KB)

显示每个进程占用了大约100M的内存，假设机器内存为32G，可拿出16G用于Apache。代入公式，得出

       apache_max_process_with_good_perfermance = 16 * 1024 * 1024 /
       108468*2=309

       apache_max_process  = 309 * 1.5=464

所以MaxClients可以设置为464

**MaxRequestsPerChild**

MaxRequestsPerChild是仅次于MaxClients的重要的配置，这个值大了影响资源的释放，小了apache不断的fork新的进程，增加CPU资源的开销。有时其影响不限于apache本身的资源，还有相关外部资源的释放。比如网站大多采用apache
pool管理数据库连接，MaxRequestsPerChild过大就会造成数据库连接的资源迟迟不释放，给数据库带来不寻常的压力。

关于这一点，最典型的就是数据库出现tomieout等超时连接错误，重启Apache以后，即使访问量恢复到峰值，数据库仍然表现毫无压力，但是持续较长时间后，又出现同样的超时异常。

我采取一分钟pv/MaxClients得到这个值。

**查看优化结果**

可以实时查看apache线程状态来查看优化结果，执行指令

    netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

结果：

    CLOSE_WAIT 530
    ESTABLISHED 232
    FIN_WAIT2 35
    TIME_WAIT 3

**Apche TCP连接状态描述**：

       CLOSED：无连接是活动的或正在进行

       LISTEN：服务器在等待进入呼叫

       SYN_RECV：一个连接请求已经到达，等待确认

       SYN_SENT：应用已经开始，打开一个连接 ESTABLISHED：正常数据传输状态 FIN_WAIT1：应用说它已经完成

       FIN_WAIT2：另一边已同意释放

       ITMED_WAIT：等待所有分组死掉

       CLOSING：两边同时尝试关闭

       TIME_WAIT：另一边已初始化一个释放

       LAST_ACK：等待所有分组死掉

由于设置了keep-alive，且时间过长，为2分钟，所以上述结果有很多的CLOSE\_WAIT进程。我们也可以使用上述的结果作为优化的输入。

## 进程内存优化

-   优化方法一

**除掉多余的模块**

    ps aux|grep -v grep|awk '/httpd/{sum+=$6;n++};END{print sum/n}'

108468

结果有100M之多，这是不大正常的，所以我尝试检查加载的module

    apachectl -t -D DUMP_MODULES

Loaded Modules:

core\_module (static)

mpm\_prefork\_module (static)

http\_module (static)

so\_module (static)

auth\_basic\_module (shared)

auth\_digest\_module (shared)

...

Syntax OK

剔除无用的模块后

       Loaded Modules:

       core_module (static)

       mpm_prefork_module (static)

       http_module (static)

       so_module (static)

       authz_host_module (shared)

       log_config_module (shared)

       expires_module (shared)

       headers_module (shared)

       setenvif_module (shared)

       mime_module (shared)

       autoindex_module (shared)

       vhost_alias_module (shared)

       negotiation_module (shared)

       dir_module (shared)

       alias_module (shared)

       proxy_module (shared)

       proxy_balancer_module (shared)

       proxy_http_module (shared)

       proxy_ajp_module (shared)

       proxy_connect_module (shared)

       cache_module (shared)

       disk_cache_module (shared)

       asis_module (shared)

       dnssd_module (shared)

       Syntax OK

重启后，显示内存102468，没有明显变化。

-   优化方法二

**MaxRequestsPerChild**

上述的优化，情况没有明显改善，对于一个仅有小部分的图片、js与小量数据查询的应用，一个进程平均占用102468的内存不合理。联想到MaxRequestsPerChild可能有影响，别忘了，除非进程处理request的数量超过这个值，不然资源不会被释放。查看http.conf。确定这个值为0，将这个值改为400。光滑重启

    apachectl -k graceful

再次查看内存，仅有12916.8。
