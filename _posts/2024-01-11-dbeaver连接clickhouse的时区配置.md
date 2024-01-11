---
layout: post
title:  "dbeaver连接clickhouse的时区配置"
date:   2024-01-11 10:29:20 +0800
categories:
      - 杂项
tags:
      - dbeaver
      - 时区配置
---

dbeaver连接clickhouse的时区问题


这几天需要使用clickhouse, 分析硬件日志。
日志已经成功导入，进行分析时发现一个问题：怎么时间不正确？少了8小时。

然后在网上找答案，方案都说如下：
- 右键->编辑连接->驱动属性
- 修改`use_server_time_zone=true`
- 修改`use_server_time_zone_for_dates=true`
- 修改`use_time_zone=Shanghai`
然后重启，即可生效。

聪明的看官已经看出来了，我这里根本不行。

又去google一圈，方案如下：
在恢复上面操作后再进行以下操作，
`窗口->首选项->用户界面` 这里有个时区配置。 设置为`Asia/Shanghai (UTC+08:00)`

然后重启软件, 即可！ 这下就真正解决了。


