---
layout: post
title:  "Browsersync启动成功但不能访问的问题"
date:   2024-12-03 17:29:20 +0800
categories:
      - 前端
      - php
tags:
      - nodejs
      - webpack
      - brosersync
---


一个php项目，使用了 Browsersync + webpack + browser-sync-webpack-plugin 技术栈。

很简单的一个项目，启动后提示成功，但是打开网页就是一直在加载中。也没有超时什么的。

![image](https://github.com/user-attachments/assets/b7583154-62e5-45d6-baa8-132e2fd30716)

就像上面的图片一样，打开链接就是不能访问。使用telnet能够连上端口，随便发送点什么会有http响应。

但是如果请求主页，就卡死。我擦。

首先这里科普下zh---
layout: post
title:  "Browsersync启动成功但不能访问的问题"
date:   2024-12-03 17:29:20 +0800
categories:
      - 前端
      - php
tags:
      - nodejs
      - webpack
      - brosersync
---


一个php项目，使用了 Browsersync + webpack + browser-sync-webpack-plugin 技术栈。

很简单的一个项目，启动后提示成功，但是打开网页就是一直在加载中。也没有超时什么的。

![image](https://github.com/user-attachments/assets/b7583154-62e5-45d6-baa8-132e2fd30716)

就像上面的图片一样，打开链接就是不能访问。使用telnet能够连上端口，随便发送点什么会有http响应。

但是如果请求主页，就卡死。我擦。

首先这里科普下 Browsersync 这个插件，他是用来同步浏览器操作的，比如在chrome上打开一个网页，安卓上打开一个，苹果上打开一个，那么你在chrome上的操作可以直接映射到其他平台

也就是同步操作。这里原理很简单：就是这个插件作为代理，在html代码中植入事件处理，然后再将事件广播到其他客户端。

那么我这里一直卡死是什么问题呢？

首先打开配置中的debug模式，在debug模式中看了下，启动日志输出：

```text
[Browsersync] [debug] Proxy running, proxing: {magenta:http://127.0.0.1:8055}
[Browsersync] [debug] Running mode: PROXY
[Browsersync] [debug] +  Step Complete: Starting the Server
[Browsersync] [debug] -> Starting Step: Starting the HTTPS Tunnel
```

这里显示是代理模式，也就是实际请求将会转发到 `http://127.0.0.1:8055` 这个地址。 

然后打开这个地址：`http://127.0.0.1:8055`， 果然打不开。 这也就为啥打开browsersync的地址也打不开网页了。因为原网页都没有启动！！

接着启动源网页，果然什么都正常了。

说实话，这个插件是真的坑。讲道理请求这种地址时这种不应该有个超时或者特定错误吗？ 

他这样真的很误导人，使用telnet能访问到端口， telnet随便发送请求会http响应失败，但是telnet发送实际http请求时却卡死无响应也没有超时时间。


好了，总结完毕！
