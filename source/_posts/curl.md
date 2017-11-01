---
title: 使用 curl 解放人肉请求url
date: 2015-08-03 15:21:01
tags:
---

最近到了一个 Java API 的项目中，看到前后端连调时，前端刷新页面发个请求，后端改一改代码，编译，上线。发现不对，再来一遍。发现名字大小写错了，前端改一改，编译(make)，发布上线。就这样能搞一晚上，人生的美好年华都在做人肉 curl 了...

所以想写文章简单介绍一下 curl 的正确姿势，解放部分蛮荒时期的人类:

通过 curl，你可以直接在命令行里发送网络请求，查看/保存 response

### GET

`curl http://klam.cc`

![code image](https://oiw32lugp.qnssl.com/2016-12-27-172442.jpg)

想要查看 response 的 header 可以增加一个参数:

`curl -i http://klam.cc`

![code image](https://oiw32lugp.qnssl.com/2016-12-27-172504.jpg)

将 response 保存到指定文件中可以用 `-o`:

`curl -o res.txt http://klam.cc`

curl 还支持设置 cookie 和 header

`curl --header "Content-type: application/json; charset=utf-8" http://klam.cc`

`curl --cookie "name=klam&gender=male" http://klam.cc`

### POST

`curl --data "name=klam&gender=male" http://klam.cc`
