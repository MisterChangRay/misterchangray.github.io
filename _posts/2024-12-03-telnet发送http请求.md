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

今天在解决一个http服务问题的时候，突发奇想：tlenet可以发送http请求吗？ 

这里直接说答案，可以：因为http是文本协议的
贴上代码：
```text
GET /test HTTP/1.0
Host: 127.0.0.1


```

以上代码修改下请求路径，在telnet连上端口后，直接复制粘贴即可发送http get 请求，注意后面的两个空行！

![image](https://github.com/user-attachments/assets/fb76b14f-7f1c-4188-983f-1ef4ad2cca90)



这里简单说下这个想法产生的原因：

上面说到http服务一个问题，其实也不是问题，主要是这个服务表现很奇怪：telnet端口探测是开放的，但是使用浏览器访问却会被阻塞，也就是一直卡在加载页面，没有超时，永远等待！

我是用telnet探测端口是开放的，然后telnet随便敲点输入，服务端也可以返回http响应。

如图：

![image](https://github.com/user-attachments/assets/b4362ea3-40bb-40e7-aff9-02b3f664f62c)


但如果我使用http协议访问则阻塞，一直等待。这里奇怪点在于，你说他不是http服务把，你不符合http协议的请求会响应http错误，但是你请求符合http协议的内容他又卡死。TMD

后来搞清楚了，原来是因为那个端口的服务是一个代理服务，如果请求的代理路径，则会转发。这里有个蛋痛的地方是：如果被代理服务没有启动，那么代理将会一直阻塞等待！！

这个真坑，搞了一下午。
