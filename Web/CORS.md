# Same-origin Policy

那么什么是Same-origin Policy呢?
简单地说, 在一个浏览器中访问的网站不能访问另一个网站中的数据,
除非这两个网站具有相同的Origin, 也即是拥有相同的协议,主机地址以及端口.
一旦这三项数据中有一项不同,
那么该资源就将被认为是从不同的Origin得来的，进而不被允许访问。

# 解决方案

为了解决这个问题, 提出了一系列方案 CORS 是其中一个.

* 更改document.domain属性
* 跨文档消息
* JSONP
* CORS

这些都是解决这个问题的方案.

# CORS

当JS发送AJAX请求访问外域的资源时, 在返回结果前,
浏览器会查看服务器返回的 response.
如果存在 Access-Control-Allow-Origin 头部. 则分析头部内容,
如果包含当前域, 浏览器就直到服务器允许这个跨域访问, 不再用同源政策限制.
将返回结果交给JS代码.

CORS将跨域访问请求分为三种:
Simple Request
Preflighted Request (预检请求)
Request with Credential(凭证)

## Simple Request

如果一个request没有包含任何自定义请求头,
而且它所使用的HTTP动词是 GET, HEAD或POST之一,
那么它被视为一个Simple Request.
但是作为POST请求时 request 的Content-type需要是
application/x-www-form-urlencoded, multipart/form-data, 或text/plain之一.

# Preflighted Request(预检请求)

如果request包含任何自定义的请求头, 或者它使用的GET, HEAD,POST之外的方法,
那么它就是 Preflighted Request.
并且 POST 请求的Content-type不是
application/x-www-form-urlencoded, multipart/form-data, 或text/plain之一.
那么它也是Preflighted Request.



