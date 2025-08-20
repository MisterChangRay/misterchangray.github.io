---
layout: post
title:  "telnet发送http请求"
date:   2024-12-03 17:49:20 +0800
categories:
      - linux
      - telnet
tags:
      - http请求
---

telnet工具，作为一个后端开发人员肯定很熟悉了。

一般使用场景是用来探测指定端口是否开发。

今天在解决一个http后台服务问题的时候，突发奇想：tlenet可以发送http请求吗？ 

这里直接说答案，可以：因为http是文本协议的
贴上代码：
```text
GET /test HTTP/1.0
Host: 127.0.0.1


```

以上代码修改下请求路径，在telnet连上端口后，直接复制粘贴即可发送http get 请求，注意后面的两个空行！

![image](https://github.com/user-attachments/assets/fb76b14f-7f1c-4188-983f-1ef4ad2cca90)



这里简单说下这个想法产生的原因：

上面说到一个后台http服务问题，具体表现在这个服务很奇怪：telnet端口探测是开放的（一般开放就认为http服务正常），但是使用浏览器访问却会被无限阻塞，也就是一直卡在加载页面，没有超时，永远等待！

接着我用telnet探测端口，是正常开放的。又使用telnet随便敲点输入，服务端也可以正常响应http请求。

如图：

![image](https://github.com/user-attachments/assets/b4362ea3-40bb-40e7-aff9-02b3f664f62c)


但我使用浏览器或者则postman访问接口就会阻塞，一直等待。这里奇怪点在于，你说他不是http服务吧，不符合http协议的请求会响应http错误，但是你请求符合http协议的内容他又卡死。TMD

后来搞清楚了，大乌龙事件！原来是那个端口的服务是一个http代理服务，如果请求的代理路径，则会转发。反之则不会。

这也就是为什么上面浏览器请求一直被阻塞，因为如果被代理服务没有启动，那么代理将会一直阻塞等待！！

这个真坑，搞了一下午。
