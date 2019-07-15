title: 记录一次 Https 握手故障的排查
categories: Coding
tags: [Https]
toc: false
date: 2018-03-08 17:58:36
---



之前开发过一款小巧的智能硬件设备，它会在用户点击的时候调用服务端的一个 RESTful API 完成下单功能。最近突然有用户反馈不能正常下单，检查服务端日志发现并没有下单的请求进入，经过排查最终发现是 Https 握手失败导致，下面记录一下详细的解决过程以加深对 Https 和 SSL/TLS 的认识。<!-- more -->

### 排查过程

经过硬件的同事检查发现网络正常，但是 https 握手失败，日志如下所示：

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803121109_837.png?imageView2/1/w/2231/h/160)

关于 `-0x7200` 错误码，查到定义如下，说明在建立 Https 阶段客户端收到了一个非法的响应，握手都没有成功，但是为什么握手失败依然不知道原因。并且通过浏览器或者 `postman` 都可以正常的访问接口，这说明服务端的 tls 配置并没有什么问题。另外在网上也找到了相关的解释以及一些讨论的 [帖子](https://tls.mbed.org/discussions/bug-report-issues/handshake-error-an-invalid-ssl-record-was-received-error-7200)， 依然不能定位到问题。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803121128_84.jpg)



后来又找了硬件的同事加了更详细的 log，重新请求终于发现了问题所在，返回的报文长度超过了最大值。原来是因为小硬件的存储资源本身非常有限，硬件端对报文大小做了限制，最大不超过 4429 字节，而从日志可以清楚的看到 handshake 返回的报文长度达到 4437，正好超过了设置的最大值，因此导致握手失败。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803121146_691.jpg)



接下来就需要分析为什么握手报文会超长，因为这个系统已经上线很久了，硬件端也没有过改动，那说明握手报文确实发生了变化，这个应该和服务端的 Https 配置有关系。之前写过的一篇 [Nginx配置自签名证书支持https服务](//jverson.com/2017/03/02/https/) 文章中有对 Https 的原理及握手过程做过一些描述，现在我们需要知道异常到底出现在哪个阶段。这时抓包神器 `Wireshark` 便可以发挥作用了。

从抓包的数据中很容易就能发现 certificate 阶段下发的报文长度是上面日志中的 4437，那应该就是在这个阶段出现了问题，再详细看看报文中的数据，光证书的长度就占了 4430 字节，可以看到总共下发了三级证书。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803121218_49.png)

隐约感觉应该是证书这里和之前不一样了，于是对公司其它的系统同样的抓包数据来对比一下如下图所示，发现主站握手时的 certificate 报文长度只有2772，仔细一看证书链只有两级，并没有下发上面的 root CA。看来就是这个原因了，经过咨询网络组同事，原来是他们在 proxy 层近期做过一些改动，增加了 root CA，但是大部分系统都把 Root CA 放到了 CDN 上，而我们这个系统还是随握手报文一起下发就导致了报文长度超过限制握手失败，最终出现了不能下单的现象。最后网络组同事更改了代理层的配置问题就解决了。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803132006_732.png)

### 证书链及服务端配置说明

关于证书链及服务端配置在网上找到一篇文章 [SSL/TLS 握手优化详解](//blog.jobbole.com/94332/) 有比较详细的解释。

TLS 的身份认证是通过证书信任链完成的，浏览器从站点证书开始递归校验父证书，直至出现信任的根证书。通过上面的抓包我们可以看到站点证书是在 TLS 握手阶段，由服务端发送的。

在配置服务端证书链时，有两点需要注意：
1. 证书是在握手期间发送的，由于 TCP 初始拥塞窗口的存在，如果证书太长可能会产生额外的往返开销；
2. 如果证书没包含中间证书，大部分浏览器可以正常工作，但会暂停验证并根据子证书指定的父证书 URL 自己获取中间证书。这个过程会产生额外的 DNS 解析、建立 TCP 连接等开销，非常影响性能。

配置证书链的最佳实践是**只包含站点证书和中间证书，不要包含根证书，也不要漏掉中间证书**。大部分证书都是「站点证书 – 中间证书 – 根证书」这样三级，这时服务端只需要发送前两个证书即可。但也有的证书有四级，那就需要发送站点证书外加两个中间证书了。不过**理想的证书链应该控制在 3kb 以内**。



### Https 握手过程抓包详解

下面通过 wireshark 抓包数据详细梳理一下 SSL/TLS 的握手过程，完整的流程如图所示

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803132016_106.png)



通过对一次 Https 请求的抓包可以清晰的看到这个过程中涉及到多次通信过程如截图所示，下面就每次通信内容逐一进行解释。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803132018_274.png)



#### 一、ClientHello

首先，客户端先向服务器发出加密通信的请求，在这一步，客户端主要向服务器提供（不限于）以下信息。

1. 支持的协议版本，比如TLS 1.0版。
2. 一个客户端生成的随机数，稍后用于生成"对话密钥"。
3. 支持的加密方法，比如RSA公钥加密。
4. 支持的压缩方法。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191728_75.png)


#### 二、SeverHello

服务器收到客户端请求后的回应包含以下内容

1. 确认使用的加密通信协议版本，如图可以看到是 **TLS1.2**
2. 一个服务器生成的随机数，稍后用于生成"对话密钥"
3. 确认使用的加密方法，从截图可以看到使用的是 **椭圆曲线（ECDHE）算法** 作为密钥交换算法

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191732_34.png)

#### 三、Certificate &  Server Key Exchange （椭圆曲线（ECDHE）算法有这一步，如果使用RSA算法则没有这一步）

Certificate 即服务器在响应 ServerHello 的同时会向 Client 端发送证书信息，上文讨论的握手失败就发生在这一步，抓包如下

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191740_558.png)



**Server Key Exchange** 这一步包括下面的 **Client Key Exchange** 都是为了得到生成会话秘钥的第三个随机参数，由于这个随机数需要加密传输避免被第三方截取，因此采用了下图所示的 [**迪菲-赫尔曼密钥交换**（英语：Diffie–Hellman key exchange，缩写为D-H）](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)。 采用DH算法后，Premaster secret不需要传递，双方只要交换各自的参数，就可以算出这个随机数。详细的过程可以参考阮一峰的 [图解SSL/TLS协议](//www.ruanyifeng.com/blog/2014/09/illustration-ssl.html) 。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191807_10.png)



#### 四、Client Key Exchange & Change Cipher Spec & Encrypted Handshake Message

客户端收到服务器回应以后，首先验证服务器证书。如果证书由可信机构颁布，客户端就会从证书中取出服务器的公钥。然后，向服务器发送下面三项信息。

1. 一个随机数。该随机数用服务器公钥加密，防止被窃听。
2. 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送。
3. 客户端握手结束通知，表示客户端的握手阶段已经结束。这一项同时也是前面发送的所有内容的hash值，用来供服务器校验。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191828_455.png)



#### 五、Server 端 Change Cipher Spec & Encrypted Handshake Message

服务器收到客户端的第三个随机数之后，计算生成本次会话所用的"会话密钥"（利用之前的三个随机数共同生成）。然后向客户端最后发送下面信息。

1. 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送。
2. 服务器握手结束通知，表示服务器的握手阶段已经结束。这一项同时也是前面发送的所有内容的hash值，用来供客户端校验。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191833_879.png)

#### 六、传送应用数据

至此，整个握手阶段全部结束。接下来，客户端与服务器进入加密通信，就完全是使用普通的HTTP协议，只不过用"会话密钥"加密内容。

![](//
jverson.oss-cn-beijing.aliyuncs.com/201803191904_387.png)



### 后续

这里由一个问题排查引出了很多的知识点，该篇只是顺便梳理一下 Https 的握手过程，但是发现随着深入，有更多的知识点需要深入挖掘，由于篇幅限制，以下的内容后续将不断追加：

1. 对话密钥的生成，这里涉及到对话秘钥的生成过程及 [Diffie-Hellman算法](//zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2%EF%BC%8D%E8%B5%AB%E5%B0%94%E6%9B%BC%E5%AF%86%E9%92%A5%E4%BA%A4%E6%8D%A2) 的使用
2. session 的复用，这里涉及到 TLS 的性能优化






### 参考

- [SSL/TLS协议运行机制的概述](//www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
- [SSL/TLS 握手优化详解](//blog.jobbole.com/94332/) 
- [图解SSL/TLS协议](//www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)