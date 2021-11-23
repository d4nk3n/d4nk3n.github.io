---
title: "“国际一流大学”"
categories: 
  - "Computer"
tags: 
  - "domain name"
---

今天在某乎上刷到[这个问题](https://www.zhihu.com/question/354444082)：

![](/uploads/img/posts/2020-02-25-first-class-universities/idn-01.jpg)

2020-02-25-first-class-universities

点开其中一个链接[国际一流大学.com](http://国际一流大学.com)，确实指向了某工业大学：

![](/uploads/img/posts/2020-02-25-first-class-universities/idn-02.jpg)

## 中文域名的原理

国际化域名实际上是将中文等Unicode字符（当然也包括“😂”）通过Punycode转换成ASCII码来访问的一种域名机制。在DNS服务器中，像`国际一流大学.com`是转换成`xn--4gqx2tswbz7c7w9a7h0d.com`来储存的。域名前面都有字符串`xn--`和普通域名相区别。

类似与将二进制字节码转化为字符串的Base64，Punycode将Unicode编码转换成ASCII码。不过它们的原理不太一样：Base64是一种基于查表的转换方法，而Punycode是Bootstring算法的一个[规范](https://tools.ietf.org/html/rfc3492)，基于一种有限状态自动机，比较复杂。具体过程可以参考这个[在线加密解密网站](http://cryptii.com)和上面提到的规范了解。

## 域名注册时间

查看whois，得益于GPDR，隐藏了注册者信息：

```bash
$ whois xn--4gqx2tswbz7c7w9a7h0d.com
   Domain Name: XN--4GQX2TSWBZ7C7W9A7H0D.COM
   Registry Domain ID: 2495908344\_DOMAIN\_COM-VRSN
   Registrar WHOIS Server: whois.dynadot.com
   Registrar URL: http://www.dynadot.com
   Updated Date: 2020-02-23T15:24:22Z
   Creation Date: 2020-02-23T11:19:46Z
   Registry Expiry Date: 2021-02-23T11:19:46Z
   Registrar: DYNADOT, LLC
   Registrar IANA ID: 472
   Registrar Abuse Contact Email: abuse@dynadot.com
   Registrar Abuse Contact Phone: +16502620100
   Domain Status: clientTransferProhibited https://icann.org/epp#clientTransferProhibited
   Name Server: NS1.DYNADOT.COM
   Name Server: NS2.DYNADOT.COM
   DNSSEC: unsigned
   URL of the ICANN Whois Inaccuracy Complaint Form: https://www.icann.org/wicf/
>>> Last update of whois database: 2020-02-26T13:29:27Z <<<
```

注册时间是2月23号，就是这两天。

## 跳转的原理

由于窝工网站不用https也能访问，会不会是通过CNAME实现的呢？
看了一下DNS没有CNAME记录，A记录指向境外服务器：

```bash
$ dig ANY xn--4gqx2tswbz7c7w9a7h0d.com

; <<>> DiG 9.11.3-1ubuntu1.7-Ubuntu <<>> ANY xn--4gqx2tswbz7c7w9a7h0d.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7128
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;xn--4gqx2tswbz7c7w9a7h0d.com.  IN      ANY

;; ANSWER SECTION:
xn--4gqx2tswbz7c7w9a7h0d.com. 242 IN    SOA     ns1.dynadot.com. hostmaster.xn--4gqx2tswbz7c7w9a7h0d.com. 1582732538 16384 2048 1048576 2560
xn--4gqx2tswbz7c7w9a7h0d.com. 37 IN     NS      ns1.dynadot.com.
xn--4gqx2tswbz7c7w9a7h0d.com. 37 IN     NS      ns2.dynadot.com.
xn--4gqx2tswbz7c7w9a7h0d.com. 37 IN     A       34.202.122.77
xn--4gqx2tswbz7c7w9a7h0d.com. 37 IN     A       35.169.225.248
xn--4gqx2tswbz7c7w9a7h0d.com. 37 IN     A       52.0.7.30

;; Query time: 34 msec
;; SERVER: 114.114.114.114#53(114.114.114.114)
;; WHEN: Thu Feb 27 00:35:39 CST 2020
;; MSG SIZE  rcvd: 196
```

使用开发者工具，发现是使用302跳转：

![](/uploads/img/posts/2020-02-25-first-class-universities/idn-03.jpg)

显然是这个域名服务商提供了301/302跳转的机制。

## 参考链接

另外，这里有个[网站](https://xn--rhqr3ykwbxv0c.top/)记录了很多类似的网址，从源代码的注释来看，大多数都是23号注册的。

这个[知乎问题](https://www.zhihu.com/question/373951041)里记录了事件的时间线。
