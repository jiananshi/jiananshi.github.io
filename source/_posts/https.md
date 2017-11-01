---
title: HTTPS 证书原理
date: 2017-06-11 09:05:08
tags: 
  - https
  - certificate
---

大家都说 HTTPS 传输是安全的，因为是基于 SSL 的，不过无论是基于任何协议，都有可以被中间人（MITM）攻击，甚至上一篇文章的代理也可以称做是中间人，因此 HTTPS 还涉及到了一个概念：证书。

访问一个支持 https 的网站即可查看它的证书信息：

![certificate](http://oiw32lugp.qnssl.com/2017-06-11-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-11%20%E4%B8%8A%E5%8D%888.32.07.png)

证书是由 CA 颁发的，这套体系是基于公私钥，通常浏览器内部会内置合法流行的 CA 公钥，而私钥自然是存在服务器端。证书的签发和验证原理如下图：

![ca](http://oiw32lugp.qnssl.com/2017-06-11-ca.png)

CloudFlare 这张图描述了在每一次请求时 HTTPS 验证的步骤：

![theory](http://oiw32lugp.qnssl.com/2017-06-11-theory.jpg)

基本上可以概括为客户端把数据同一些随机字符通过公钥加密后发送给服务端，服务端接收后使用私钥揭秘后通过这些数据源生成一个 session key，客户端和服务端持有同样的 session key 后通信。

CA 从组织上分为 root Certificates 和 intermediates Certificates，它们都可以颁发证书，通常客户端（浏览器）仅需保存 root Certificates，而 intermediates Certificates 的信息则是通过**证书链**一层一层向上验证：

![chain](http://oiw32lugp.qnssl.com/2017-06-11-chain.gif)

了解了这些原理后，我们尝试自己签发一个证书，不过这在实际生活中并不适用，因为个人自签证书并且推广到各大客户端预存是不可能的事，除非做到 Let’s encrypt 这种级别。

`node` 下有一个 TLS 协议的实现可以帮助我们：

```js
const forge = require('node-forge');
const pki = forge.pki;
const path = require('path');

var keys = pki.rsa.generateKeyPair(1024);
var cert = pki.createCertificate();
cert.publicKey = keys.publicKey;
cert.serialNumber = (new Date()).getTime() + '';

// 设置CA证书有效期
cert.validity.notBefore = new Date();
cert.validity.notBefore.setFullYear(cert.validity.notBefore.getFullYear() - 5);
cert.validity.notAfter = new Date();
cert.validity.notAfter.setFullYear(cert.validity.notAfter.getFullYear() + 20);
var attrs = [{
    name: 'commonName',
    value: 'ymy'
}, {
    name: 'countryName',
    value: 'CN'
}, {
    shortName: 'ST',
    value: 'Shanghai'
}, {
    name: 'localityName',
    value: 'Shanghai'
}, {
    name: 'organizationName',
    value: 'ymy'
}, {
    shortName: 'OU',
    value: 'https://yemengying.com'
}];
cert.setSubject(attrs);
cert.setIssuer(attrs);
cert.setExtensions([{
    name: 'basicConstraints',
    critical: true,
    cA: true
}, {
    name: 'keyUsage',
    critical: true,
    keyCertSign: true
}, {
    name: 'subjectKeyIdentifier'
}]);

// 用自己的私钥给CA根证书签名
cert.sign(keys.privateKey, forge.md.sha256.create());

var certPem = pki.certificateToPem(cert);
var keyPem = pki.privateKeyToPem(keys.privateKey);

console.log('公钥内容：\n');
console.log(certPem);
console.log('私钥内容：\n');
console.log(keyPem);
```

如果在服务端配置过 HTTPS 证书会很容易看出这段代码会创建一个公钥和私钥，我们可以在脚本里把它写到 `.crt` 文件中就可以被客户端安装了，现在出于为了理解原理的目的我们仅仅只是输出了它。

站点支持 HTTPS 虽然在协议级别会有更多的信息传输，更多的「握手」🤝，但是总的来说是利大于弊，最明显的一点就是不会被运营商插广告了，记得曾经积分商城页面被用同样的 HTML 结构将礼品列表替换成了情色内容诱惑点击量还被用户投诉了，当时看到用户的截图时第一反应是：还有这种操作？

![operation](http://oiw32lugp.qnssl.com/2017-06-10-20170502154345_31928.jpg)