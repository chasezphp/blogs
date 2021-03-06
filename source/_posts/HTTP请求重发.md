---
title: HTTP请求重发
date: 2016-07-06
tags:
- HTTP
---

HTTP 协议中，从语义上讲， GET 请求一般是获取服务器端的资源，不会对服务器数据造成副作用，可简单理解为一种“读”操作；而 POST 请求多用于更改（增、删、改）服务器上的资源，会产生一定的副作用。

所以，这样看起来，浏览器是不是就不会因为网络原因啥的自动重发 POST 请求吧？实际上是这样么？
<!-- more -->

# 起因

最近在对接地图的一个数据录入接口：前端向后端发送一个 CSV 文件，后端将 CSV 文件中的数据解析出来，然后将数据通过地图接口导入到地图数据库。由于地图提供的接口有点怪异，批量导入数据的接口有一些问题，只能使用单条导入接口，所以在这里， CSV 文件里面有多少条数据，就会访问多少次地图的接口。

虽然有点坑，不过问题究竟是解决了，于是就这样上线了。

天有不测风云，遇到一个客户，一下要导入上千条数据，后端这样串行地一条一条去导入，很轻易地就花了好几分钟。而且还遇到一个诡异的现象：每条数据都导入了两次！

# 分析

凭借多年的前端开发经验（不要脸了），立马大胆猜测，浏览器发送了两次请求。

于是先到谷歌开发者工具的 Network 标签页检查一下请求，发现此处只记录了一次请求，并且该请求没有响应，好像看不出来什么猫腻。再切换到 chrome://net-internals/ 中看看日志，发现一个 error code ， google 了一下，并没有什么结果，看起来也验证不了猜想。

然后再去找后端同学看看接口日志，是不是访问了两次，后端同学似乎稍微有点不太想打日志重新部署（过程比较麻烦），所以先放弃用这种方式求证。

那用啥求证呢？ Charles 吧。

在 Charles 中一看，发现发了四次请求，每次请求基本上都耗时六十多秒，每次都没有响应内容。

好了，看起来就是浏览器六十秒超时重发请求。

# 深入分析

可以转念一想，这对么？

- 1、 POST 请求就这样轻易地被浏览器超时重发，难道浏览器开发者没考虑过数据重复发送的问题吗？表单 POST 请求手动刷新浏览器的时候都会弹窗提醒用户要不要重复提交数据呢！
- 2、为啥是六十秒呢？时间这么短吗？想想平时本地断点调试服务器代码的时候，那可是会超时老长时间的，所以这六十秒算个啥呢？

搜一搜往上资料，发现 [HTTP/1.1 的一处规范](https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html#sec8.2.4) ：

> If an HTTP/1.1 client sends a request which includes a request body, but which does not include an Expect request-header field with the "100-continue" expectation, and if the client is not directly connected to an HTTP/1.1 origin server, and if the client sees the connection close before receiving any status from the server, the client SHOULD retry the request.

大致意思就是说，如果发送一个请求到服务器端，该请求有请求体，但是请求头里面不包含“ 100-continue ”这种东西，并且客户端没有直接连接到原始的 HTTP/1.1 服务器，此时，如果客户端在接收到服务器发送的 HTTP 状态之前发现服务器主动关掉连接，那么客户端应该重试请求。

那看起来好像就是服务器端主动关掉了连接，导致浏览器重新发送请求了。

我们服务器端使用的是 Tomcat ，查一查资料，发现 Tomcat 默认的 connector 超时时间是六十秒，刚好吻合上了。

问题原因找到了，解决起来就轻松了，此处不赘述。
