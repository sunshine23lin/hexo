---
title: http核心技术
date: 2020-12-15 10:03:03
categories: 计算机网络
tags: 计算机网络
---

##  前言

- **HTTP 基本概念**
- **Get 与 Post**
- **HTTP 特性**
- **HTTP 与 HTTPS**
- **HTTP/1.1、HTTP/2、HTTP/3 演练**

![image-20201215140430600](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215140430600.png)

##  HTTP基本概念

 ###  初入认识

**http是超文本传输协议，是在一个计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范**。

### 状态码

![image-20201215141652119](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215141652119.png)

###  常见字段

- **Host**

  客户端发送请求时，用来指定服务器的域名

  ![image-20201215141938002](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215141938002.png)

- **Content-Length** 

  服务器在返回数据时，会有 `Content-Length` 字段，表明本次回应的数据长度。

  ![image-20201215143529802](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215143529802.png)

  如上面则是告诉浏览器，本次服务器回应的数据长度是 1000 个字节，后面的字节就属于下一个回应了。

-  **Connection**

  `Connection` 字段最常用于客户端要求服务器使用 TCP 持久连接，以便其他请求复用。

  ![image-20201215143627738](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215143627738.png)

  HTTP/1.1 版本的默认连接都是持久连接，但为了兼容老版本的 HTTP，需要指定 `Connection` 首部字段的值为 `Keep-Alive`。

-  **Content-Type**

  `Content-Type` 字段用于服务器回应时，告诉客户端，本次数据是什么格式。

  ![image-20201215143719801](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215143719801.png)

  ~~~http
  Content-Type: text/html; charset=utf-8
  ~~~

  上面的类型表明，发送的是网页，而且编码是UTF-8。

  ```http
  Accept: */*
  ```

  上面代码中，客户端声明自己可以接受任何格式的数据。

- **Content-Encoding**

  `Content-Encoding` 字段说明数据的压缩方法。表示服务器返回的数据使用了什么压缩格式

  ~~~http
  Content-Encoding: gzip
  ~~~

  上面表示服务器返回的数据采用了 gzip 方式压缩，告知客户端需要用此方式解压。

  客户端在请求时，用 `Accept-Encoding` 字段说明自己可以接受哪些压缩方法。

  ```http
  Accept-Encoding: gzip, deflate
  ```

##  GET和POST

###  GET

`Get` 方法的含义是请求**从服务器获取资源**，这个资源可以是静态的文本、页面、图片视频等

比如，你打开我的文章，浏览器就会发送 GET 请求给服务器，服务器就会返回文章的所有文字及资源。

![image-20201215144359800](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215144359800.png)

###  POST

而`POST` 方法则是相反操作，它向 `URI` 指定的资源提交数据，数据就放在报文的 body 里。

比如，你在我文章底部，敲入了留言后点击「提交」，浏览器就会执行一次 POST 请求，把你的留言文字放进了报文 body 里，然后拼接好 POST 请求头，通过 TCP 协议发送给服务器。

![image-20201215144458204](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215144458204.png)

###  GET 和 POST 区别？

####  安全幂等

先说明下安全和幂等的概念：

- 在 HTTP 协议里，所谓的「安全」是指请求方法不会「破坏」服务器上的资源
- 所谓的「幂等」，意思是多次执行相同的操作，结果都是「相同」的。

那么很明显 **GET 方法就是安全且幂等的**，因为它是「只读」操作，无论操作多少次，服务器上的数据都是安全的，且每次的结果都是相同的。

**POST** 因为是「新增或提交数据」的操作，会修改服务器上的资源，所以是**不安全**的，且多次提交数据就会创建多个资源，所以**不是幂等**的。

####  参数存放

- GET请求参数是放在URL后面，从而容易导致被攻击者窃取，对你的信息超成破坏和伪造，**对URL有长度限制**；
- POST请求参数是放在请求体BODY中。对数据长度没有要求。

####  TCP数量

- get 请求在发送过程中会产生一个 TCP 数据包，浏览器会把 http header 和 data 一并发送出去，服务器响应 200（返回数据）
- post 在发送过程中会产生两个 TCP 数据包，浏览器先发送 header，服务器响应 100 continue，浏览器再发送 data，服务器响应 200 ok（返回数据）。

## HTTP特性

###  优点

- 简单
- 灵活和易于扩展
- 应用广泛跨平台

###  缺点

HTTP 协议里有优缺点一体的**双刃剑**，分别是「无状态、明文传输」，同时还有一大缺点「不安全」。

- **无状态双刃剑**

  无状态的**好处**，因为服务器不会去记忆 HTTP 的状态，所以不需要额外的资源来记录状态信息，这能减轻服务器的负担，能够把更多的 CPU 和内存用来对外提供服务。

  无状态的**坏处**，既然服务器没有记忆能力，它在完成有关联性的操作时会非常麻烦

  **解决方法：使用cookie技术**

  `Cookie` 通过在请求和响应报文中写入 Cookie 信息来控制客户端的状态。

  相当于，**在客户端第一次请求后，服务器会下发一个装有客户信息的「小贴纸」，后续客户端请求服务器的时候，带上「小贴纸」，服务器就能认得了了**

  ![image-20201215145522022](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215145522022.png)

  ![image-20201215145537027](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215145537027.png)

  

- **明文传输双刃剑**

  明文意味着在传输过程中的信息，是可方便阅读的，通过浏览器的 F12 控制台或 Wireshark 抓包都可以直接肉眼查看，为我们调试工作带了极大的便利性。

  但是这正是这样，HTTP 的所有信息都暴露在了光天化日下，相当于**信息裸奔**。在传输的漫长的过程中，信息的内容都毫无隐私可言，很容易就能被窃取，如果里面有你的账号密码信息。

- **不安全**

  使用HTTPS解决。

##  HTTPS

HTTP 由于是明文传输，所以安全上存在以下三个风险：

- **窃听风险**，比如通信链路上可以获取通信内容，用户号容易没。
- **篡改风险**，比如强制入垃圾广告，视觉污染，用户眼容易瞎。
- **冒充风险**，比如冒充淘宝网站，用户钱容易没。

HTTPS 在HTTP与TCP层之间加入SSL/TLS协议

![image-20201215150223967](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215150223967.png)

可以很好的解决了上述的风险：

- **信息加密**：交互信息无法被窃取，但你的号会因为「自身忘记」账号而没。
- **校验机制**：无法篡改通信内容，篡改了就不能正常显示，但百度「竞价排名」依然可以搜索垃圾广告。
- **身份证书**：证明淘宝是真的淘宝网，但你的钱还是会因为「剁手」而没。

**HTTPS是如何解决上面的三个风险？**

- **混合加密**的方式实现信息的**机密性**，解决了窃听的风险。
- **摘要算法**的方式来实现**完整性**，它能够为数据生成独一无二的「指纹」，指纹用于校验数据的完整性，解决了篡改的风险。
- 将服务器公钥放入到**数字证书**中，解决了冒充的风险。

1. 混合加密

   通过**混合加密**的方式可以保证信息的**机密性**，解决了窃听的风险。

   ![image-20201215150715241](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215150715241.png)

   HTTPS 采用的是**对称加密**和**非对称加密**结合的「混合加密」方式：

   - 在通信建立前采用**非对称加密**的方式交换「会话秘钥」，后续就不再使用非对称加密。
   - 在通信过程中全部使用**对称加密**的「会话秘钥」的方式加密明文数据。

   采用「混合加密」的方式的原因：

   - **对称加密**只使用一个密钥，运算速度快，密钥必须保密，无法做到安全的密钥交换。
   - **非对称加密**使用两个密钥：公钥和私钥，公钥可以任意分发而私钥保密，解决了密钥交换问题但速度慢。

2. 摘要算法

   **摘要算法**用来实现**完整性**，能够为数据生成独一无二的「指纹」，用于校验数据的完整性，解决了篡改的风险。

   ![image-20201215150813968](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215150813968.png)

   客户端在发送明文之前会通过摘要算法算出明文的「指纹」，发送的时候把「指纹 + 明文」一同
   加密成密文后，发送给服务器，服务器解密后，用相同的摘要算法算出发送过来的明文，通过比较客户端携带的「指纹」和当前算出的「指纹」做比较，若「指纹」相同，说明数据是完整的。

3. 数字证书

   客户端先向服务器端索要公钥，然后用公钥加密信息，服务器收到密文后，用自己的私钥解密。

   这就存在些问题，如何保证公钥不被篡改和信任度？

   所以这里就需要借助第三方权威机构 `CA` （数字证书认证机构），将**服务器公钥放在数字证书**（由数字证书认证机构颁发）中，只要证书是可信的，公钥就是可信的。

   ![image-20201215150946771](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215150946771.png)



##  HTTP/1.1、HTTP/2、HTTP/3 演变

###  说说 HTTP/1.1 相比 HTTP/1.0 提高了什么性能？

HTTP/1.1 相比 HTTP/1.0 性能上的改进：

- 使用 **TCP 长连接**的方式改善了 HTTP/1.0 短连接造成的性能开销。
- 支持 管道（pipeline）网络传输，只要第一个请求发出去了，不必等其回来，就可以发第二个请求出去，可以减少整体的响应时间。

HTTP/1.1 还是有性能瓶颈：

- 请求 / 响应头部（Header）未经压缩就发送，首部信息越多延迟越大。只能压缩 `Body` 的部分；
- 发送冗长的首部。每次互相发送相同的首部造成的浪费较多；
- 服务器是按请求的顺序响应的，如果服务器响应慢，会招致客户端一直请求不到数据，也就是队头阻塞；
- 没有请求优先级控制；
- 请求只能从客户端开始，服务器只能被动响应。

那上面的 HTTP/1.1 的性能瓶颈，HTTP/2 做了什么优化？

HTTP/2 协议是基于 HTTPS 的，所以 HTTP/2 的安全性也是有保障的。

###    HTTP/2 相比 HTTP/1.1 性能上的改进

1. **头部压缩**

   HTTP/2 会**压缩头**（Header）如果你同时发出多个请求，他们的头是一样的或是相似的，那么，协议会帮你**消除重复的分**。

   这就是所谓的 `HPACK` 算法：在客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就**提高速度**了。

2. **二进制格式**

   HTTP/2 不再像 HTTP/1.1 里的纯文本形式的报文，而是全面采用了**二进制格式。**

   头信息和数据体都是二进制，并且统称为帧（frame）：**头信息帧和数据帧**。

   ![image-20201215152141511](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215152141511.png)

   这样虽然对人不友好，但是对计算机非常友好，因为计算机只懂二进制，那么收到报文后，无需再将明文的报文转成二进制，而是直接解析二进制报文，这**增加了数据传输的效率**。

3. **数据流**

   HTTP/2 的数据包不是按顺序发送的，同一个连接里面连续的数据包，可能属于不同的回应。因此，必须要对数据包做标记，指出它属于哪个回应。

   每个请求或回应的所有数据包，称为一个数据流（`Stream`）。

   每个数据流都标记着一个独一无二的编号，其中规定客户端发出的数据流编号为奇数， 服务器发出的数据流编号为偶数

   客户端还可以**指定数据流的优先级**。优先级高的请求，服务器就先响应该请求。

   ![image-20201215152242276](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215152242276.png)

4. **多路复用**

   HTTP/2 是可以在**一个连接中并发多个请求或回应，而不用按照顺序一一对应**。

   移除了 HTTP/1.1 中的串行请求，不需要排队等待，也就不会再出现「队头阻塞」问题，**降低了延迟，大幅度提高了连接的利用率**。

   举例来说，在一个 TCP 连接里，服务器收到了客户端 A 和 B 的两个请求，如果发现 A 处理过程非常耗时，于是就回应 A 请求已经处理好的部分，接着回应 B 请求，完成后，再回应 A 请求剩下的部分。

   ![image-20201215152403683](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215152403683.png)

5. **服务器推送**

   HTTP/2 还在一定程度上改善了传统的「请求 - 应答」工作模式，服务不再是被动地响应，也可以**主动**向客户端发送消息。

   举例来说，在浏览器刚请求 HTML 的时候，就提前把可能会用到的 JS、CSS 文件等静态资源主动发给客户端，**减少延时的等待**，也就是服务器推送（Server Push，也叫 Cache Push）。

###  HTTP/2 有哪些缺陷？HTTP/3 做了哪些优化？

HTTP/2 主要的问题在于：多个 HTTP 请求在复用一个 TCP 连接，下层的 TCP 协议是不知道有多少个 HTTP 请求的。

所以一旦发生了丢包现象，就会触发 TCP 的重传机制，这样在一个 TCP 连接中的**所有的 HTTP 请求都必须等待这个丢了的包被重传回来**。

- HTTP/1.1 中的管道（ pipeline）传输中如果有一个请求阻塞了，那么队列后请求也统统被阻塞住了
- HTTP/2 多请求复用一个TCP连接，一旦发生丢包，就会阻塞住所有的 HTTP 请求。

这都是基于 TCP 传输层的问题，所以 **HTTP/3 把 HTTP 下层的 TCP 协议改成了 UDP！**

![image-20201215152550236](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215152550236.png)

UDP 发生是不管顺序，也不管丢包的，所以不会出现 HTTP/1.1 的队头阻塞 和 HTTP/2 的一个丢包全部重传问题。

大家都知道 UDP 是不可靠传输的，但基于 UDP 的 **QUIC 协议** 可以实现类似 TCP 的可靠性传输。

- QUIC 有自己的一套机制可以保证传输的可靠性的。当某个流发生丢包时，只会阻塞这个流，**其他流不会受到影响**。
- TL3 升级成了最新的 `1.3` 版本，头部压缩算法也升级成了 `QPack`。
- HTTPS 要建立一个连接，要花费 6 次交互，先是建立三次握手，然后是 `TLS/1.3` 的三次握手。QUIC 直接把以往的 TCP 和 `TLS/1.3` 的 6 次交互**合并成了 3 次，减少了交互次数**

![image-20201215152614617](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215152614617.png)

所以， QUIC 是一个在 UDP 之上的**伪** TCP + TLS + HTTP/2 的多路复用的协议。

QUIC 是新协议，对于很多网络设备，根本不知道什么是 QUIC，只会当做 UDP，这样会出现新的问题。所以 HTTP/3 现在普及的进度非常的缓慢，不知道未来 UDP 是否能够逆袭 TCP。

##  无状态协议

无状态协议就是指浏览器对于事务的处理没有记忆能力。例如比如客户请求获得网页之后关闭浏览器，然后再次启动浏览器，登陆该网站，但是服务器并不知道客户关闭了一次浏览器。

HTTP 就是一种无状态的协议，他对用户的操作没有记忆能力。可能大多数用户不相信，他可能觉得每次输入用户名和密码登陆一个网站后，下次登陆就不再重新输入用户名和密码了。这其实不是 HTTP 做的事情，起作用的是一个叫做 `小甜饼(Cookie)` 的机制。它能够让浏览器具有`记忆`能力。

如果你的浏览器允许 cookie 的话，查看方式 **chrome://settings/content/cookies**

![image-20201215172321303](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215172321303.png)

当你向服务端发送请求时，服务端会给你发送一个认证信息，服务器第一次接收请求时，开辟了一块Session空间（创建了Session对象）

同时生成一个sessionID,并通过响应头的Set-Cookie: JSESSIONID=XXXXXXX 命令，向客户端发送要求设置Cookie的响应；客户端接收响应后，在本机客户端设置一个JSESSIONID=XXXXXXX 的 Cookie 信息，该 Cookie 的过期时间为浏览器会话结束。

![image-20201215173131296](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215173131296.png)

​    接下来客户端每次向同一个网站发送请求时，请求头都会带上该Cookie信息（包含sessionid）,然后，服务器通过读取请求头中cookie信息，获取名称为JSESSIONID 的值，得到每次请求SessionID，这样，你的浏览器才具有记忆能力。

![image-20201215173524513](https://jameslin23.gitee.io/2020/12/15/http核心技术/image-20201215173524513.png)

​     还有一种方式是使用 JWT 机制，它也是能够让你的浏览器具有记忆能力的一种机制。与 Cookie 不同，JWT 是保存在客户端的信息，它广泛的应用于单点登录的情况。JWT 具有两个特点

- JWT 的 Cookie 信息存储在`客户端`，而不是服务端内存中。也就是说，JWT 直接本地进行验证就可以，验证完毕后，这个 Token 就会在 Session 中随请求一起发送到服务器，通过这种方式，可以节省服务器资源，并且 token 可以进行多次验证。
- JWT 支持跨域认证，Cookies 只能用在`单个节点的域`或者它的`子域`中有效。如果它们尝试通过第三个节点访问，就会被禁止。使用 JWT 可以解决这个问题，使用 JWT 能够通过`多个节点`进行用户认证，也就是我们常说的`跨域认证`。

