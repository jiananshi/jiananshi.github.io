---
title: 小巧好用的压力测试工具 wrk
date: 2017-06-24 13:31:21
tags:
  - shell
  - 工具
---

上周服务被打挂了一次，发现的时候 CPU + Memory 几乎被占满，但是请求并发量只有 ~50 左右。修复 + 重启以后一直在想平时怎么在日常自测服务，压力测试需要联系公司专门的团队 + 排期并不十分方便，佳浩老师给推荐了 wrk，这里记录一下使用心得。

wrk 的[官网](https://github.com/wg/wrk)关于用法介绍的挺详细的，这里就只简单举一个例子：

```bash
wrk -t4 -c100 -d10s -T30s --latency https://shijianan.com
```

这条命令的意思是：起四个线程，并发量为 100，持续发送 10 秒钟，HTTP 的超时设置为 30s。10 秒后 wrk 将会返回一个结果的报告，其中涉及到一些指标我也花了段时间弄懂：

```bash
Running 10s test @ https://shijianan.com
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   722.48ms  297.44ms   2.63s    91.92%
    Req/Sec    37.96     27.98   150.00     72.13%
  Latency Distribution
     50%  626.69ms
     75%  645.88ms
     90%  861.00ms
     99%    2.12s
  1302 requests in 10.08s, 6.41MB read
Requests/sec:    129.16
Transfer/sec:    650.83KB
```

这份测试结果报告中比较容易理解的是 Avg 和 Max，即请求的平均响应时间及最大响应时间，但是只看这两个指标一定是不靠谱的，左耳朵耗子也曾经在他的博客 [性能测试应该怎么做？ | | 酷 壳 - CoolShell](http://coolshell.cn/articles/17381.html) 中说到过平均值并没有代表性，因此我们还要关注一个指标：Stdev

Stdev(Standard Deviation)即标准偏差，是统计学的一个名词，这里表示请求响应时间的离散程度，值越大代表请求响应时间的差距越大，系统的响应约不稳定。

最后 Latency Distribution 是指响应时间的分布情况，即 50% 的请求在 0ms ~ 626.69ms，而高于 2.12s 的请求只占 1%（HOST 这个网站的服务器只有一台在日本，CDN 会快很多，优化的空间还是很大的）。

最后 +/- Stdev 这个指标怎么都没搜到，还请知道详情的读者在评论或邮件告知。