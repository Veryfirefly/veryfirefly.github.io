---
layout: page
title: 关于我
permalink: /about/
---

# 联系方式

- Email：<a href="mailto:993610942@qq.com">993610942@qq.com</a>
- QQ/微信号：<a target="_blank" href="http://wpa.qq.com/msgrd?v=3&uin=993610942&site=qq&menu=yes">993610942</a>
- website : [http://www.likeu.cool/](http://www.likeu.cool/ "http://www.likeu.cool/")

---

# 个人信息

 - 最红/男/1997 
 - 工作年限：3.5年
 - 技术博客：[https://veryfirefly.github.io](https://veryfirefly.github.io "https://veryfirefly.github.io")
 - Github：[https://github.com/veryfirefly](https://github.com/veryfirefly "https://github.com/veryfirefly")
 - Gitee：[https://gitee.com/GangHuo_admin](https://gitee.com/GangHuo_admin "https://gitee.com/GangHuo_admin")
 - 期望职位：Java高级程序员
 - 期望薪资：税前月薪**12k-15k**
 - 期望城市：成都

---

# 工作经历

## 成都聚乐科技有限公司 （2017年12月 - 至今）

### 1. 2018年4月-2018年7月  点球大战（H5小游戏）

该项目为2.5D的点球大战小游戏，对接**IGXE平台**接入该游戏，玩家可在平台中兑换CS:GO的皮肤等。

在该项目中遇到的问题和解决思路及该项目对我的能力提升：

1. 初期采用Adobe Animate编写，无法模拟玩家踢出球之后，足球的运行轨迹及下降轨迹。于是我另辟蹊径使用了Cannon.js（WebGL物理引擎）负责实现游戏中的物理效果模拟，并采用Three.js实现Graphics Render，及为2D图片的足球绑定一个3D不可视的球形物理碰撞。
2. chrome由于安全策略的问题无法自动播放音频。我是用了Sound.js加载音频，由于chrome自动播放音频的策略问题，只有在玩家加载完游戏后点击进入游戏时自动播放音频。
3. 在该项目中，我还使用了Tween.js实现加载动画及过场动画。
4. 在该项目中，我使用的是原生Servlet进行开发进球率及判断球是否成功射进的后端逻辑及后台数据管理页面。
5. 在该项目中，我负责对接IGXE，对接IGXE提供的**兑入兑出接口**、**用户日志上报接口**、**游戏内跳转商城链接接口**，并负责为其提供**用户信息查询接口**、**查询账号金币接口**、**进球率控制接口**等

### 2. 2018年7月-至今 游戏管理后台

1. 我在这个项目中负责维护并优化原有功能的使用速度，并改善项目结构，根据策划需求调整和快速做出新功能。
2. 由于在该项目中，游戏内的数据日志全部上报到后台所独立出来的日志工程中，每日产生数据量高达上百万，做分表处理后，单表数据量上千万，严重影响查询速度，并造成mysql占用cpu资源过高、tomcat会将请求等待或直接丢弃不处理。我采取了：<u>优化sql、优化表索引、对某些很耗的查询功能进行请求排队，防止请求次数过高造成宕机的问题、对基础数据在日志上报中就进行一级处理，避免在查询时计算处理数据</u>。成功将多个查询时间需要数分钟的功能优化到5s内。

### 游戏内相关项目 

1. 我在这个项目负责与运营渠道对接，获取他们所需要的功能后快速定制api接口，并负责维护该工程。
2. 解决了游戏内部系统在浏览器中无法正常拉起客户端的问题(外部应用调用)，以及自定义协议在https下出现的不安全问题。


### h5小游戏项目

1. 我在这个项目中与策划交流，负责编写2.5D的点球大战的小游戏，配合IGXE接入小游戏。在该项目中，我遇到过2d游戏不能模拟球的自由下落的问题。最后我采用<u>Three.js + Cannon.js</u> 库来编写，由<u>Cannon.js来负责实现物理引擎</u>，而使用<u>Three.js则提供Graphics Render</u>，并采用了<u>Tween.js来解决UI动画</u>、及使用<u>Sound.js作为游戏中的音效实现</u>。

### 其他项目


1. 公司官网项目，我们采用了自研架构来解决分布式问题，使用JRaft来做一致性协调工作。

---

# 开源项目和作品

- 在今年3月份加入了Alibaba Open Source，并成为了<u>Apache RocketMQ</u>、<u>FastJson</u>、<u>Nacos</u> 的 <u>Contributor</u>
- 分析阅读Apache Tomcat源码中，了解Tomcat的<u>容器组件</u>、<u>生命周期</u>、<u>以及Connector对请求的处理</u>，尝试并<u>成功从源码中构建Tomcat</u>
- 分析阅读MyCat-NIO源码中，了解并学习<u>mysql 客户端和服务端之间的通信协议</u>

# 技能清单

以下均为我熟练使用的技能

- Web开发：Java/Servlet
- Web框架：SpringBoot/Spring-Framework/Mybatis
- 分布式框架：Dubbo/SpringCloud
- 前端框架：Bootstrap/HTML5/ECMAScript5/TypeScript
- 数据库相关：MySQL/Redis/InfluxDB
- 版本管理工具：Svn/Git
- 单元测试：Junit
- 云和开放平台：QQ开放平台/微博开放平台
- linux：centos

# 自我评价

热爱钻研新兴架构和技术，并热爱混迹开源社区从中尝试提升自己，喜欢学习。对工作充满热情，踏实有耐心。

# 致谢

感谢您花时间阅读我的简历，期待能有机会和您共事。