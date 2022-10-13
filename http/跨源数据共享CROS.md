### 跨域资源共享

**跨域资源共享**（CROS）是一种基于 HTTP 头的机制，该机制通过允许服务器标示除了自己以外的其他origin(域，协议和端口)，使得**浏览器**允许这些origin访问加载自己的资源。

跨源资源共享还通过一种机制来检查服务器是否允许发送真实的请求，该机制通过**浏览器**发起一个到服务器**托管**的跨域资源的"预检"请求。**在预检中，浏览器发送的头中标有HTTP方法和真正请求中会用到的头**。



**跨源请求的一个例子：**运行在 https:A.com 的JavaScript 代码使用 XMLHttpRequest 来发起一个到 https:B.com 的一个请求。

出于安全性，浏览器限制脚本内发起跨源的 HTTP 请求。

跨源域资源共享(CROS) 机制允许**Web应用服务器**进行跨源访问控制，从而使跨源数据传输得以安全进行。



##### HOW

跨源资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。另外，规范要求，对那些可能对服务器数据产生副作用的 HTTP 请求方法(特别是 GET 以外的HTTP请求，或者搭配某些 MIME类型的POST请求)，浏览器必须首先使用 OPTIONS 方法发起一个预检请求 (preflight request)，从而获知服务端是否允许该跨域请求。服务器运行之后，才发起时机的HTTP请求。在预检请求的返回中，服务端也可以通知客户端，是否需要携带身份凭证（包括 Cookies和HTTP认证相关数据）。



##### HTTP响应首部字段

**Access-Control-Allow-Origin**

响应首部中可以携带一个 Access-Control-Allow-Origin 字段，其语法如下：

```
Access-Controll-Allow-Origin:<origin>|*
```

其中，origin 参数的值指定了运行访问资源的外域URL。对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符，表示允许来自所有域的请求。

例如，下面的字段值将允许来自 https://fenchuiyun.com 的请求

```
Access-Controll-Allow-Origin: https://fenchuiyun.com
Vary: Origin
```

如果服务器指定了具体的域名而非  "*",那么响应首部中的 Vary 字段的值必须包含 Origin。这将告诉客户端：服务器对不同的源站返回不同的内容。



**Access-Control-Expose-Headers**

Access-Control-Expose-Header 头让服务器把允许浏览器访问的头放入白名单，例如：

```
Access-Controll-Expose-Headers: X-My-Custom-Header,X-Another-Custom-Header
```

这样浏览器就能够通过 getResponseHeader 访问 X-My-custom-Header 和 X-Another-Custom-Header 响应头了。



**Access-Control-Max-Age**

Access-Control-Max-Age 头指定了preflight 请求结果能够被缓存多久

```
Access-Control-Max-Age:<delta-seconds>
```

**delta-seconds** 参数表示preflight 预检请求的结果在多少秒内有效。



等

[跨源资源共享(CROS)]: https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#http_%E5%93%8D%E5%BA%94%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5

