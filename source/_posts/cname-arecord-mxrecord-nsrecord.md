---
title: A 记录、CNAME、NS 记录和 MX 记录介绍
date: 2017-01-03 22:19:37
tags:
---

## A 记录

A 记录将域名指到 IP 地址，用户可以直接访问你的域名而无需去记你的 IP，略微有规模的站点都会部署多个集群和灾备环境，通过配置多条 A 记录可以实现负载均衡和容灾

```bash
PING shijianan.com  
>> 64 bytes from 104.28.10.81: icmp_seq=0 ttl=54 time=274.181 ms
```

可以看到 shijianan.com 的 IP 是 104.28.10.81，正是 A 记录的配置将 IP 地址和域名对应起来。如果你希望将任何三级域名指向根域，可以将 IP 通过 A 记录映射到 \*.shijianan.com，这样无论是 www.shijianan.com 还是 abc.shijianan.com 都会访问 shijianan.com，这叫做 **泛域名解析**。

## CNAME

CNAME  可以理解为域名的别名，比如 shijianan.com 有一天换名字了 jiananshi.com，但是我们已经通过二维码将网址宣传出去了，线下回收和重新宣传的成本太高了，可以通过使用 CNAME 将 shijianan.com 重定向到 jiananshi.com（前提是 shijianan.com 依然为你所有）。同一个主机地址，CNAME 是不能和 A 记录共存的，A 记录的优先级要高于 CNAME。

```bash
dig shijn.github.io
>> ;; ANSWER SECTION:
>> shijn.github.io. 3600  IN  CNAME github.map.fastly.net.
```

可以看到 github 将我的域名重定向到了 github.map.fastly.net

## NS 记录

NS（Name Server）指定那台服务器来对指定域名解析，通常是在你的域名购买商的地方配置，我们购买域名后的第一件事通常都是设置 NS 记录到我们的主机服务商（Linode、DigitalOcean、阿里云或者提供中间服务的 CloudFlare），这个记录至少要有两条。同一主机地址，NS 记录优先级高于 A 记录。

## MX 记录

MX 记录用来配置邮件服务的地址，比如：

```bash
shijianan.com MX 10 mail.shijianan.com
```

数字 10 代表优先级，MX 记录可以配置多条，需要注意的是：MX 记录优先级数字越小优先级越高。
