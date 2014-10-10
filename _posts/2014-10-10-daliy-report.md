---
layout: post
title: daily report(2014年10月)
category: knowledge
tags: [pyinotify]
---

# 2014-10-09
今天完成:

* Region 模块的抽象,但是编译未通过,问题比较奇怪;
* 和OP,QA 进行case study, 沟通10月7日的问题原因;
* 定位线上superfile2的问题,通过sauron确定问题为bmq发送消息失败,原因为第三套集群压力过大;
* 目前superfile2的topic增加至24个,第三组topic偶尔压力大;
 
明天计划:

* 完成Region请求编译,功能基本通过;
* 增加validate的判定;
* 和OP过监控方案;

