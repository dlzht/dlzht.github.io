+++
title = "Let's Encrypt计划终止OCSP服务"
date = 2024-07-25
[taxonomies]
tags = ["杂文"]
+++

今天看到了Let's Encrypt计划将终止OCSP服务, 转而启用CRL服务的文章[Intent to End OCSP Service](https://letsencrypt.org/2024/07/23/replacing-ocsp-with-crls.html). 想起了当年Let's Encrypt宣布提供免费的3个月证书, 一改域名证书垄断高价的局面, 再加上犀利的[acme.sh](https://github.com/acmesh-official/acme.sh)脚本, 可以说是造福了无数个人开发者和站长. 今天也写一写OCSP的内容, 重温一下那时岁月.

<!-- more -->

## 数字证书

了解OCSP和CRL之前, 我们先简单了解下数字证书, Let's Encrypt主要是签发的域名证书, 我们这边也以域名证书为例.

{{ image(src="/image/013_01.png", alt="image 404", position="center") }}

首先, 只有权威机构CA颁发的数字证书, 我们的浏览器才会默认信任(可以手动添加). 数字证书里包含了CA的数字签名, 是无法伪造和篡改的.

其次, 数字证书里包含域名和密钥信息, CA签发出来了证书, 就相当于在做证明, 拥有这个域名控制权的人, 也拥有这个密钥的控制权.

{{ image(src="/image/013_02.png", alt="image 404", position="center") }}

我们可以直接在浏览器里看网站的证书, 比如用Firefox打开`github`, 点击地址栏旁边的小锁, 选择`安全连接 -> 更多信息 -> 查看证书`.

在TLS握手环节中, 数字证书可以证明, 对面和我们握手的, 确实是这个域名下的服务器, 而不是什么中间人假冒的, 数字证书是互联网安全中非常重要的一环.

## OCSP和CRL

[OCSP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol)(在线证书状态协议)和[CRL](https://en.wikipedia.org/wiki/Certificate_revocation_list)(证书吊销列表)是CA提供的"公共"服务, 目地在于提供实时查询证书的状态的能力. 上面的`github`证书信息里的`AIA`一项, 就告诉了我们应该去哪里查询这张证书的OCSP和CRL信息.

证书本身是包含了有效期的, 说明了证书在什么时间段是有效的. 但还有其他的情况需要让证书失效, 比如说客户的私钥失窃了, 或者CA一是疏忽发错了证书, 或者某些证书存在算法漏洞等等.

所以CA提供了证书吊销的机制, 尽管证书此时可能还在有效期内, 但已经不能被信任了. 已经发出去的证书文件没办法保证全部收回的, 所以CA要提供实时查询证书状态的服务.

{{ image(src="/image/013_03.png", alt="image 404", position="center") }}

当向CA发起CRL查询的时候, CA返回的是一个吊销列表, 里面包含了很多被吊销的证书. 我们要验证某张证书是不是被吊销了, 就看这张证书在不在这个列表中. 


{{ image(src="/image/013_04.png", alt="image 404", position="center") }}

而发起OCSP请求的时候, 我们指明要查询的哪张证书, CA也就返回这张证书的状态. 当然, 这边返回状态不止已吊销或未吊销, 还有OCSP服务内部错误, 请求数据有问题等等状态.

## OCSP的优劣

1. OCSP服务段隐私安全问题, 当用户向CA提交OCSP请求的时候, OCSP服务就知道了用户要访问的是什么网站. 出于一些原因, 服务端可能会主动或被迫地记录下相关信息, 这对用户来说就存在隐私风险.

2. 网络传输过程中信息泄露, 因为OCSP协议没有强制加密传输, 所以用户向OSCP服务端发起查询时, 通过网络监听或者中间人攻击等手段, 域名这样的信息就可能泄露了.

3. OCSP的优势在于只查询某张证书, 所以响应包会比较小, 解析起来也更容易. PDF数字签名开启LTV时, 也可以把OCSP响应写入到签名数据中, 而不会导致文件体积暴增.

## OCSP装订(Stapling)

OCSP装订解决了OCSP存在的一些的问题, 简单来说, 就是服务器会缓存OCSP查询的结果, 然后当客户端来TLS握手时, 就合在证书里一起发出去, 这样另一端就不需要再去查询OCSP了.

这样的好处是, 一方面, 省去了客户端查询OCSP的时间, 用户可以更快地得到响应; 另一方面, 客户端不需要和OCSP服务交互, 也就没有了上面的隐私安全问题, 还减少了网络问题带来的不可用(比如某个CA突然被屏蔽了).

nginx里开启OCSP Stapling:
```text
server {
  ssl_stapling on;
  ssl_stapling_verify on;
}
```

CloudFlare提供的支持: [https://blog.cloudflare.com/high-reliability-ocsp-stapling](https://blog.cloudflare.com/high-reliability-ocsp-stapling)

## 终止OCSP服务

停止OCSP服务的一个原因, Let's Encrypt在文章里提到, 一个是资源开销问题, 在2023年的另一篇文章里有提到, 当时其服务的域名就超过了3亿, 每秒需要响应10万OCSP请求, 无疑这需要的计算和带宽资源都不小.

虽然OCSP装订在某种程度上可以缓解CA的负担, 但Staplig需要站长侧那边开启, CA侧并没有主动权. 不知道是不是Stapling带来的效果并不够, 导致从资源方面考虑还是偏向了CRL方案.

> Even when a CA intentionally does not retain this information, as is the case with Let’s Encrypt, CAs could be legally compelled to collect it. CRLs do not have this issue

另一个原因是隐私泄露, 在现今审核越来越严, 限制越来越多的局势下, 真的要保护用户隐私, "做不到"确实也是一种没办法的办法. 我提供的是CRL, 我不知道用户要访问的是什么网站, 从技术上我没办法知道的, 是这样的.

至于Let's Encrypt为什么要采用CRL方案, 其中原因也不揣测, 就这样了. CRL也一样能用的, 而且现在的浏览器对证书验证都有容错机制, 所以对于web服务的影响应该不会太大.

</br>

*-> 如果文章有不足之处或者有改进的建议，可以在[这边](https://github.com/dlzht/dlzht.github.io/discussions/12)告诉我，也可以发送给我的[邮箱](mailto:dlzht@protonmail.com)*