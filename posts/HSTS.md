---
title: HSTS in Https
date: 2019-08-07 10:32:29
tags: 
- http
- https
---

## 问题

今天中午发现之前的一个内网地址(`a.cc.oa.com`)访问不了了,问了下同事,被告知解决办法如下：

- chome地址框中输入: `chrome://net-internals/#hsts`

  ![chrome-hsts](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-01-13-142125.png)

- 把域名`cc.oa.com`、`a.cc.oa.com`分别delete

好了,恢复正常! 继续工作...结果没过一会又不能访问了。

## 原因

之前做CDN的时候,多少接触过HSTS,索性看看啥问题吧。

### 为什么会有HSTS

在chrome会把http协议的网站标记为`Not Secure`的背景下,各类网站都开始了全面https的支持,那么用户输入域名之后,如何切换到https协议呢? 让用户手动输入`https`肯定是不可接受的。

很多网站采用302跳转的方式支持http到https的转换。

#### 302跳转

以访问`www.baidu.com`为例(复现需要按上述步骤在`chrome://net-internals/#hsts`页面删除`www.baidu.com`)

##### Round 1st - HTTP

- Request

```http
GET / HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Chrome/75.0.3770.142 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9,zh-CN;q=0.8,zh
```

- Response

```http
HTTP/1.1 302 Found
Connection: Keep-Alive
Content-Length: 225
Content-Type: text/html
Date: Tue, 06 Aug 2019 14:16:24 GMT
Location: https://www.baidu.com/
Server: BWS/1.1
Set-Cookie: BD_LAST_QID=16606; path=/; Max-Age=1
X-Ua-Compatible: IE=Edge,chrome=1
```

##### Round 2nd - HTTPS

- Request 

```http
GET / HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Chrome/75.0.3770.142 Safari/537.36
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9,zh-CN;q=0.8,zh
```

- Response 

```http
HTTP/1.1 200 OK
Bdpagetype: 1
Bdqid: 0xae2c06010000c61c
Cache-Control: private
Connection: Keep-Alive
Content-Encoding: gzip
Content-Type: text/html
Cxy_all: baidu+b6d07af5fbdbdba08cfb8fe17af0d780
Date: Tue, 06 Aug 2019 13:50:59 GMT
Expires: Tue, 06 Aug 2019 13:50:40 GMT
Server: BWS/1.1
Set-Cookie: delPer=0; path=/; domain=.baidu.com
Set-Cookie: BD_HOME=0; path=/
Set-Cookie: H_PS_PSSID=1452; path=/; domain=.baidu.com
Strict-Transport-Security: max-age=172800
Vary: Accept-Encoding
X-Ua-Compatible: IE=Edge,chrome=1
Transfer-Encoding: chunked
```



#### 302跳转有什么问题

- 每次http请求都会有302,导致多一次跳转
- 由于Round 1st是还是明文的http,可能会被劫持

这时候**HSTS**登场了。

### 什么是HSTS

[ HTTP Strict Transport Security (HSTS)](https://tools.ietf.org/html/rfc6797) 是由[rfc6797](https://tools.ietf.org/html/rfc6797)定义的。

#### 配置实例

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

##### max-age 

有效期,单位是s

##### includeSubDomains

对所有子域名生效

##### 效果

在经历了如上Round 1st, Round 2nd后,此后的`http://www.baidu.com`浏览器会进行内部跳转(无网络访问)

```http
HTTP/1.1 307 Internal Redirect
Location: https://www.baidu.com/
Non-Authoritative-Reason: HSTS
```

#### 问题完全解决了么

虽然后续访问不会再有301/302跳转，但是首次访问仍然是301/302 http跳转的，仍有可能被劫持。

chrome维护了一个[HSTS Preload List](https://hstspreload.org/)，这个列表会被各大浏览器更新到浏览器的preload list,这样浏览器初始就支持应该对网站使用https，直接就会307，而不会再有301/302（不会因为明文传输被劫持）。

> The HSTS preload list is a set of domains that have opted into HSTS, which enforces that those domains can only be accessed over HTTPS. Once their site is ready, webmasters can submit their domain to hstspreload.org, which will result in their domain being hard-coded as HTTPS-only in Chrome’s list.
>
> Most major browsers (Chrome, [Firefox](https://blog.mozilla.org/security/2012/11/01/preloading-hsts/), Opera, Safari, [IE 11 and Edge](https://blogs.windows.com/msedgedev/2015/06/09/http-strict-transport-security-comes-to-internet-explorer-11-on-windows-8-1-and-windows-7/)) also have HSTS preload lists based on the Chrome list. 



## 解决问题

通过最终定位发现`a.cc.oa.com`不支持https访问；但是`cc.oa.com`的响应头中会包含`Strict-Transport-Security: max-age=31536000; includeSubDomains`导致访问`a.cc.oa.com`时会自动跳转到https，导致服务无法访问。

可见，HSTS一定有弄清楚才可以线上配置，否则可能会导致用户无法访问！由于**HSTS一旦上线不容易回滚**

- 设置了max-age之后，浏览器就会在max-age周期内默认跳转https，如果网站https没有ready，那就是故障...

- Removal From HSTS Preload List 可能耗费数月的时间...

  > Be aware that inclusion in the preload list cannot easily be undone. Domains can be removed, but it takes months for a change to reach users with a Chrome update and we cannot make guarantees about other browsers. Don't request inclusion unless you're sure that you can support HTTPS for **your entire site and all its subdomains** in the long term.

所以一般建议的步骤是：

1. 支持通过301/302跳转的访问；

2. 设置HSTS，max-age逐步增大；
3. 加入到[HSTS Preload List](https://hstspreload.org/)。

## Reference

[SSL labs](https://www.ssllabs.com/ssltest/)

[Let’s Encrypt](https://letsencrypt.org/)