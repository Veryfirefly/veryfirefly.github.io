---
layout: post
title: 自定义外部协议使浏览器拉起本地程序
date: 2019-11-11
Author: 来自veryfirefly
categories: 
tags: [工作经验]
comments: true
---

## 什么是自定义协议 ##

由于我们的游戏需要在浏览器中调用NPAPI插件，而chrome移除了NPAPI的支持，导致游戏并不能很好的适配所有的浏览器，所以这个时候我们对于chrome浏览器用到了自定义浏览器协议这一标准。自定义浏览器协议允许在浏览器中使用`protocol://url`的形式进行调用本地程序。包括在网页上拉起百度云网盘、或者拉起QQ等等等等，都属于自定义浏览器协议。

![调用浏览器协议示例](https://veryfirefly.github.io/images/baiduyun_example.png)

## 原理分析 ##

在通过浏览器调用外部程序时，浏览器会在我们本地的注册表中查找协议所对应的注册表，并获取实际要调用的程序路径进行调用。

> 例：`xy://callback/?id=opaqueInternalAccesssObj`

`xy://`为我们自定义的浏览器协议，后面的`callback/?id=opaqueInternalAccessObj`则为实际传入的参数(如果需要的话)

### 注册表解析 ###

	Windows Registry Editor Version 5.00

	[HKEY_CLASSES_ROOT\xy]
	@="GameLoader Plugin"
	"URL Protocol"="C:\\Windows\\System32\\cmd.exe"
	
	[HKEY_CLASSES_ROOT\xy\shell]
	
	[HKEY_CLASSES_ROOT\xy\shell\open]
	
	[HKEY_CLASSES_ROOT\xy\shell\open\command]
	@="C:\\Windows\\System32\\cmd.exe \"%1\""

保存成`xy.reg`，双击运行后在浏览器中输入`xy://`后会提示是否打开GameLoader Plugin(实则是打开Windows Shell)。

1. `[HKEY_CLASSES_ROOT]` 是应用程序运行时必须的信息，`[HKEY_CLASSES_ROOT\xy]`表示在该注册表目录下生成了一个xy的应用程序运行时必须的信息。`@=`为该应用程序默认名称，用来显示程序名称，不填则为exe名称， `URL Protocol=`为该协议所要调用的程序地址。

2. `[HKEY_CLASSES_ROOT\xy\shell]`在xy\下生成shell目录。

3. `[HKEY_CLASSES_ROOT\xy\shell\open]`在xy\shell\下生成open目录

4. `[HKEY_CLASSES_ROOT\xy\shell\open\command]`在xy\shell\open\下生成command目录，`@=`在command目录下新建一个默认值为协议调用程序的实际路径。

### 自定义协议的坑 ###

我们的游戏客户端注册表一开始没有在`[HKEY_CLASSES_ROOT\xy]`下写入`"URL Protocol=(path...)"`的注册信息，导致在chrome v74及以上浏览器中无法拉起外部程序，而在v74以下则可以拉起。遇到这个问题时，起初我怀疑是chrome在更新后移除掉这个功能，于是从v68版本看到了v74版本的移除功能说明文档后，发现该功能未更新也未移除。于是我困惑的想到去下个chrome v74版本来进行测试，打开百度云地址，点击下载，发现百度云网盘能拉起客户端，于是`f12`查看百度云所调用的地址`baiduyunguanjia://xxxxxxx`，卧槽？

遂进入regedit.exe中进行查看，默默的比对百度云与我们的注册表有何不同，查看到根目录时发现百度云多了个`URL Protocol`，于是在我们的注册表上加上后，chrome v74版本及以上的都可以调用外部程序了，<span style="color:red">所以不管在新版还是旧版的浏览器中，注册表中在根目录一定要加上<strong>URL Protocol</strong>的路径，以确保能正常拉起本地程序。</span>