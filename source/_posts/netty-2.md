---
title: netty-"Hello World"
date: 2019-08-02 13:29:08
categories: Netty
tags:
---
前期pom文件

```xml
 <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.8.Final</version>
 </dependency>

```

1. TestServer

```java
public class TestServer{
     /**
      * 事件循环组，就是死循环
      */
      //仅仅接受连接，转给workerGroup，自己不做处理
      EventLoopGroup bossGroup=new NioEventLoopGroup();
      //真正处理
      EventLoopGroup workerGroup=new NioEventLoopGroup();
      try {
            //很轻松的启动服务端代码
            ServerBootstrap serverBootstrap=new ServerBootstrap();
            //childHandler子处理器,传入一个初始化器参数TestServerInitializer（这里是自定义）
            //TestServerInitializer在channel被注册时，就会创建调用
            serverBootstrap.group(bossGroup,workerGroup).channel(NioServerSocketChannel.class).
                    childHandler(new TestServerInitializer());
            //绑定一个端口并且同步，生成一个ChannelFuture对象
            ChannelFuture channelFuture=serverBootstrap.bind(8899).sync();
            //对关闭的监听
            channelFuture.channel().closeFuture().sync();
        }
        finally {
            //循环组优雅关闭
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }


 }

```
2. TestServerInitializer

```java
   public class TestServerInitializer extends ChannelInitializer<SocketChannel> {
    //这是一个回调的方法，在channel被注册时被调用
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //一个管道，里面有很多ChannelHandler，这些就像拦截器，可以做很多事
        ChannelPipeline pipeline=ch.pipeline();
        //增加一个处理器，neet提供的.名字默认会给，但还是自己写一个比较好
        /**
         * 注意这些new的对象都是多例的，每次new出来新的对象,因为每个连接的都是不同的用户
         */
        //HttpServerCodec完成http编解码，可查源码
        pipeline.addLast("httpServerCodec",new HttpServerCodec());
        //增加一个自己定义的处理器hander
        pipeline.addLast("testHttpServerHandler",new TestHttpServerHandler());
    }
}    


```
3. TestHttpServerHandler

```java
/**
 * 继承InboundHandler类,代表处理进入的请求,还有OutboundHandler
 */
public class TestHttpServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    // channelRead0读取客户请求,并返回响应方法
    protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {
        // 判断这个是不是httpRequest
//         if (msg instanceof HttpRequest) {
            // 判断url是否请求了favicon.ico
            System.out.println(msg.getClass());
            AttributeKey<String> ATTRIBUTE_KEY_VIN = AttributeKey.valueOf("VIN");
            ctx.channel().attr(ATTRIBUTE_KEY_VIN).set("aaavv00");
            System.out.println("VIM:"+ctx.channel().attr(ATTRIBUTE_KEY_VIN).get());
            System.out.println(ctx.channel().remoteAddress());
            HttpRequest httpRequest= (HttpRequest) msg;
            URI uri=new URI(httpRequest.uri());
            // 判断url是否请求了favicon.ico
            if ("/favicon.ico".equals(uri.getPath())){
                System.out.println("请求了favicon.ico");
                return;
            }

            /**
             * 上面这段代码是验证如果用浏览器访问
             * 浏览器发起了两次请求,一个是发起的端口,第二是请求favicon.ico图标
             */

            // 代表响应返回的数据
            ByteBuf context = Unpooled.copiedBuffer("Hello World", CharsetUtil.UTF_8);
            //构造一个http响应,HttpVersion.HTTP_1_1:采用http1.1协议，HttpResponseStatus.OK：状态码200
            FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK, context);
            response.headers().set(HttpHeaderNames.CONTENT_TYPE, "test/plain");
            response.headers().set(HttpHeaderNames.CONTENT_LENGTH, context.readableBytes());
            // 如果只是调用write方法,他仅仅在存在缓冲区,并不会返回客户端
           // ctx =null;
            ctx.writeAndFlush(response);
//  }
    }


    /**
     * 添加handler
     * @param ctx
     * @throws Exception
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("handlerAdded");
        super.handlerAdded(ctx);
    }

    /**
     * 管道被注册
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        System.out.println("channelRegistered");
        super.channelRegistered(ctx);
    }

    /**
     * 当建立连接之后调用此方法
     * @param ctx
     * @throws Exception
     */
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("channelActive");
        super.channelActive(ctx);
    }


    /**
     * 断开连接调用此方法
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("channelInactive");
        super.channelInactive(ctx);
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        System.out.println("channelUnregistered");
        super.channelUnregistered(ctx);
    }

    /**
     * 当发送异常的时候调用此方法
     * @param ctx
     * @param cause
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        super.channelActive(ctx);
    }
}


```


