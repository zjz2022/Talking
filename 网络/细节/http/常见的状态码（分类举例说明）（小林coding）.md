### 常见的状态码（分类举例说明）

![图片](D:/%E6%96%87%E4%BB%B6/typora%E5%9B%BE%E7%89%87/640.webp)五大类 HTTP 状态码

- 1xx 类状态码属于**提示信息**，是协议处理中的一种中间状态，实际用到的比较少。
- 2xx 类状态码表示服务器**成功**处理了客户端的请求，也是我们最愿意看到的状态。
- 3xx 类状态码表示客户端请求的资源发生了变动，需要客户端用新的 URL 重新发送请求获取资源，也就是**重定向**。
- 4xx 类状态码表示客户端发送的**报文有误**，服务器无法处理，也就是错误码的含义。
- 5xx 类状态码表示客户端请求报文正确，但是**服务器处理时内部发生了错误**，属于服务器端的错误码。

### 301和302有什么区别

重定向状态码如下，301 和 302 都会在响应头里使用字段 Location，指明后续要跳转的 URL，浏览器会自动重定向新的 URL。

- 「**301 Moved Permanently**」表示永久重定向，说明请求的资源已经不存在了，需改用新的 URL 再次访问。
- 「**302 Found**」表示临时重定向，说明请求的资源还在，但暂时需要用另一个 URL 来访问。

重定向是指将一个URL请求转发到另一个URL的过程。重定向的作用包括：

- 更改URL：通过重定向，可以更改URL，使其更易于记忆、更友好或更有意义。例如，将长而复杂的URL重定向到简洁的、易于理解的URL。
- 网站迁移：当网站进行重构、更换域名或更改URL结构时，通过重定向旧的URL到新的URL，可以让用户和搜索引擎正确地访问和索引新的内容。