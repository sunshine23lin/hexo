---
title: OpenResty
date: 2020-12-24 08:16:32
categories: web服务器
tags: 缓存
---

##   OpenResty介绍

OpenResty(又称：ngx_openresty) 是一个基于 nginx的可伸缩的 Web 平台，由中国人章亦春发起，提供了很多高质量的第三方模块。

OpenResty 是一个强大的 Web 应用服务器，Web 开发人员可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块,更主要的是在性能方面，OpenResty可以 快速构造出足以胜任 10K 以上并发连接响应的超高性能 Web 应用系统。

360，UPYUN，阿里云，新浪，腾讯网，去哪儿网，酷狗音乐等都是 OpenResty 的深度用户。

OpenResty 简单理解成 **就相当于封装了nginx,并且集成了LUA脚本**，开发人员只需要简单的其提供了模块就可以实现相关的逻辑，而不再像之前，还需要在nginx中自己编写lua的脚本，再进行调用了。

###  安装OpenResty

linux 安装openresty

1. 添加仓库执行命令

   > yum install yum-utils
   >
   >  yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo

2. 执行安装

   > yum install openresty

3. 安装默认路径

   > /usr/local/openresty

###  安装nginx

默认已经安装好了nginx,在目录: `/usr/local/openresty/nginx`

修改`/usr/local/openresty/nginx/conf/nginx.conf`,将配置文件使用的根设置为root,目的就是将来要使用lua脚本的时候 ，直接可以加载在root下的lua脚本。

> cd /usr/local/openresty/nginx/conf
> vi nginx.conf

![image-20201224082722268](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224082722268.png)

###  测试访问

输入网页访问

![image-20201224082754599](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224082754599.png)

##  nginx+lua缓存

**如下思路:**

1. 首先访问nginx，先获取本地缓存，获取到直接响应

2. 没有获取到，再访问redis，我们可以从redis获取数据，如果有则返回响应，并且缓存到nginx中

3. 如果没有获取到，再次访问MySQL，我们从MySQL中获取数据，再将数据存储到redis中，返回

4. 而在这里面，我们都可以使用LUA脚本嵌入到程序中执行这些查询相关的业务。

   ![image-20201224083639389](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224083639389.png)

**nginx增加缓存命名空间**

`lua_shared_dict dis_cache 128m`

![image-20201224083902436](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224083902436.png)

**nginx访问调用lua脚本**

![image-20201224084101865](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201224084101865.png)

```conf
location /read_content {
     content_by_lua_file /root/lua/read_content.lua;
}
```

**编辑lua脚本**

read_content.lua

```lua
ngx.header.content_type="application/json;charset=utf8"
local uri_args = ngx.req.get_uri_args();
local id = uri_args["id"];
--获取本地缓存
local cache_ngx = ngx.shared.dis_cache;
--根据ID 获取本地缓存数据
local contentCache = cache_ngx:get('content_cache_'..id);

if contentCache == "" or contentCache == nil then
    local redis = require("resty.redis");
    local red = redis:new()
    red:set_timeout(2000)
    red:connect("192.168.10.132", 6379)
    local rescontent=red:get("content_"..id);

    if ngx.null == rescontent then
        local cjson = require("cjson");
        local mysql = require("resty.mysql");
        local db = mysql:new();
        db:set_timeout(2000)
        local props = {
            host = "192.168.10.132",
            port = 3306,
            database = "changgou_content",
            user = "root",
            password = "123456"
        }
        local res = db:connect(props);
        local select_sql = "select url,pic from tb_content where status ='1' and category_id="..id.." order by sort_order";
        res = db:query(select_sql);
        local responsejson = cjson.encode(res);
        red:set("content_"..id,responsejson);
        ngx.say(responsejson);
        db:close()
    else
        cache_ngx:set('content_cache_'..id, rescontent, 10*60);
        ngx.say(rescontent)
    end
    red:close()
else
    ngx.say(contentCache)
end
```

