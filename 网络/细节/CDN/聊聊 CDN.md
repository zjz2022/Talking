## 聊聊 CDN

想必大伙都听过 CDN（Content Delivery Network），几乎市面上所有上点规模的公司都会用到 CDN。

我前段时间看了一本《CDN技术详解》，然后本地简单实现了一个 CDN 服务，这篇文章来简单总结一下 CDN 的一些关键点。

大多数人应该都了解它的基础原理。如果网站服务器部署在北京，香港的用户访问该网站，由于物理距离的缘故（网络的传输时间受距离影响），时延相对而言会比较大，导致体验上并不是很好。

![图片](D:/%E6%96%87%E4%BB%B6/typora%E5%9B%BE%E7%89%87/640.webp)图片

并且当用户量比较大的时候，全国各地所有的请求都涌向北京的服务器，网络的主干道就会被阻塞。这个应该很好理解，就好比国庆假期的网红打卡点，大家都想去那里打卡，那么前往这个打卡点的道路不就都被堵满了吗？

这其实就是所谓的“热点”问题，该如何解决这个问题呢？可以采取类似分流的操作。

主服务器部署在北京不变，但是全国各地都建立一些缓存站，这些缓存站可以缓存一些主服务器上变化不频繁的资源，比如一些 css/js/图片等静态资源。

当香港的用户想去访问网站的时候，根据就近原则，选择一个距离它最近的缓存站，比如有个深圳站：

![图片](D:/%E6%96%87%E4%BB%B6/typora%E5%9B%BE%E7%89%87/640-1715696289689-1.webp)图片

那么先去这个站上看看有没有资源，如果有就直接就从缓存站要到资源了，这个流量就被深圳站拦截了，由于距离很近，响应时延也很低，且不占用请求北京的主干道流量，也减轻了北京服务器的负担，一举多得！

如果缓存站找不到，那么就回源到源站，也就是缓存站会去北京的服务器去要数据，然后将这些数据缓存到本地且返回给用户，用户的感觉可能就是网站卡了一下，然后就好了。

这次操作后，别的香港用户访问同样的内容由于缓存站缓存了，因此不需要再次回源，直接从深圳站就返回数据了。

当然，一些热点的数据也可以让源站主动推送给缓存站，进行缓存预热，这样减少了回源的动作，使得高峰时期服务器更加平稳，用户也不会有卡一下的表现。

同理，源站也可以主动刷新缓存来更新缓存站上的老数据，缓存站上的缓存也可以设置有效期，过期删除，减轻缓存站的存储压力。

## 动态数据怎么办？

上面我说的是静态资源，如果是提交订单这样的操作怎么缓存呢？不还是得请求北京的服务器吗？

确实如此，很多业务场景涉及到存储层面是有状态的，如果我们让缓存站也能处理这些业务场景，就得将一些业务数据存储下来，那么就又会涉及数据同步问题。

这种数据同步机制又会带来另一个高复杂度的挑战，也就是数据一致性问题，很复杂。

所以现在很多云厂商提供的全站 CDN， 一般指的是 CDN 厂商会自动识别你网站资源哪些是静态的，哪些是动态的。

静态的按照我们上面说的路子走，动态的则是根据内部的一些调度算法，智能地选择最优回源路径去请求源站，节省请求的时间。

这就好比我们自驾从香港开到北京，路线有很多，然后有个导航很智能，它可以实时监控计算当前道路信息，给我们提供一条路径最短、最不堵的路。

## CDN 基础架构

接下来我们就来分析下 CDN 究竟是如何工作的，一图胜千言，我们先来看看下图：

![图片](D:/%E6%96%87%E4%BB%B6/typora%E5%9B%BE%E7%89%87/640-1715696289690-2.webp)图片

所谓的 CDN 缓存节点就是我上面说的缓存站，然后还是一个很重要的 GSLB （全局负载均衡器）。

这个 GSLB 的主要功能就是实现我上面说的 ：`当香港的用户想去访问网站的时候，根据就近原则，选择一个距离它最近的缓存站`。

没错，它最主要的功能就是用户访问网站的时候，根据用户请求的 ip、url 选择最近的节点，让用户直接请求最近的节点即可。

简单来说就是请求转发，转发到最近（或许是最空闲）的站。

GSLB 转发机制有三种实现：

- 基于 DNS 解析
- 基于 HTTP 重定向（主流应用层协议为 HTTP）
- 基于 IP 路由。

业界实现转发的主流技术是 DNS 解析，这里大家可以花 10 秒钟思考一下，为什么主流是 DNS 解析？

答案是 DNS 有缓存功能和负载均衡能力，能天然减轻 GSLB 的压力。

设想一下，如果采用 HTTP 重定向去实现转发，从功能角度来看有问题吗？

没问题，301 永久重定向和 302 临时重定向都可以实现转发功能。

那么 301 合适吗？永久重定向好像不合适，比如重定向的缓存站挂了咋办，迁移了咋办？GSLB 叫全局负载均衡，就均衡一次完事儿啦？

所以只能 302 ，而 302 的流程每次还得访问 GSLB，那不就等于所有请求每次都得经过 GSLB 操作？因此 GSLB 可能会成为性能瓶颈。

而 DNS 解析不会这样，用户通过域名解析定位到 GSLB ，通过负载均衡返回用户最近的一个缓存站 ip，后续浏览器、本地 DNS 服务器等都会将本次域名解析得到的 ip 结果缓存一段时间。

那么这个时间段内用户再次请求这个域名，压根不会打到 GSLB 而是直接访问对应的缓存站，这不就解决瓶颈问题了吗？

> tips：以下内容需要提前你了解 DNS 基本解析流程，篇幅有限，这里不多介绍 DNS 相关知识

基于 DNS 解析具体有三种实现方式：

- 利用 CNAME 实现负载均衡
- 将 GSLB 作为权威 DNS 服务器
- 将 GSLB 作为权威 DNS 服务器的代理服务器

业界最多是使用 CNAME 方式来实现负载均衡，实现简单且不需要修改公共 DNS 系统配置。

> 利用 CNAME 如果实现 CDN？

简单举个例子，比如之前网站网址是 `www.netitv.com.cn`，此时进行要 cdn 改造，那么将之前的网站网址作为 GSLB  服务域名的 CNAME，用户访问 `www.netitv.com.cn`，经过 CNAME 解析会映射到 GSLB 地址 `www.netitv.cdn.com.cn` 上，然后 GSLB 基于 DNS 协议可以进行后续的负载均衡操作，选择合适的 IP 返回给用户。

具体如下图所示，图来自《CDN技术详解》：![图片](D:/%E6%96%87%E4%BB%B6/typora%E5%9B%BE%E7%89%87/640-1715696289690-3.webp)

第二种实现方式其实很好理解，就是将 GSLB 作为一个域的权威 DNS 服务器，取而代之，那么对于这个域来说，正常域名解析的过程不就可以为所欲为了？负载均衡就都由 GSLB 来把控了，至于如何才能成为一个域的权威 DNS 服务器？这个我不清楚，听起来好像有点难度。

至于第三种实现方式其实就是在权威 DNS 服务器前面做一个代理，差别就是不需要实现一个功能完整的权威 DNS 服务器，仅需对个别需要 GSLB 操作的请求进行修改转发即可，不过这其实也得将对外公布的权威 DNS 服务器的地址变成代理服务器地址，这个难度和第二点一致。

好了，最后还有一个 IP 路由没介绍，其实本质的原理是基于路由器原有的路由算法和数据包转发能力来实现转发。

简单来说就是域名解析得到一个 VIP（虚拟 IP），用户向 VIP 请求的时候，经过路由器 A， 路由器 A 查看路由表来进行转发数据包，然后发现有多个路由，因此就可以不同路径的转发了，一般这种只能实现在某个内部网络，全国性基本不可能实现，所以简单了解下就行。

其他还有一些我最上面图画的健康检测、异常转移等等，这个其实和正常的微服务操作都是一样的，这里就不多介绍了。

## 最后

我大概花了 7 小时读完了 《CDN技术详解》，上面就是我整理的对开发同学而言关键的一些点，有兴趣的同学也可以自行去阅读这本书。

关于自建 CDN 可以利用 Netty 实现 DNS 的解析，然后内部做一些负载均衡的操作，将电脑 DNS 服务器设为 127.0.0.1 即可调试，这里就不多做展开了。

**推荐阅读：**

[真香！想冲得物去了！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247534598&idx=1&sn=10509d797fbf49c561d5253a957b090f&chksm=f98d0caccefa85baacc4e9eff7721a7748294249f8faf382961090071b7a526946871bc38572&scene=21#wechat_redirect)**
**

[泪崩，中厂一面也要输了。。。](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247534553&idx=1&sn=1191c9fd4f802f4365df39b4b3d17779&chksm=f98d0373cefa8a65daa7075ce3485c1722bf9e8dbe79d4bb44074f6e263a22632baf9d85bb44&scene=21#wechat_redirect)**
**

[腾讯一二面，强度拉满！](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247534280&idx=1&sn=5010bb309a259ea5013445293daabbe1&chksm=f98d0262cefa8b7425c04d33f0a81cdd927f70d9f5397a5bfae12f06303b46e5af4239fb8300&scene=21#wechat_redirect)**
**

[字节都到三面了，结果还是凉了。。。](http://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247534099&idx=1&sn=06f222165c250aee1950e1bfaf689763&chksm=f98d02b9cefa8baf1c30bfcd9655d03fe80beebd99e85996915241711ad18aaa6cd2688ec6d7&scene=21#wechat_redirect)