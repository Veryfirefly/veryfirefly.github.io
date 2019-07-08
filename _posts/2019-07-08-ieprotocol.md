---
layout: post
title: 使用Chrome、Firefox等其他浏览器拉起IE浏览器
date: 2019-07-08
Author: 来自veryfirefly
categories: 
tags: [工作经验,解决方法]
comments: true
---

上周突然遇到一个很奇怪的需求，要求是使用Chrome拉起IE浏览器，这是为什么呢？

因为我们游戏是用C++写的，嵌入到浏览器中是用的**NPAPI**插件，我们内部写了一套js函数通过浏览器来拉起C盘目录下的游戏客户端，但让人烦恼的一点是Chrome 42版本下不支持NPAPI，那么我们要拉起客户端就必须使用IE内核或者是Chrome内核。

就突发奇想的使用非IE浏览器拉起IE浏览器，这并不是最终的解决办法，而只是我自己试验的一个小插曲，这里只是将该办法分享出来，给需要用到的朋友，也方便自己以后查看。

## 1.使用Chrome浏览器拉起IE浏览器

这个方法是我在 [stackoverflow](https://www.stackoverflow.com) 上找到的，是使用注册表进行URL Protocol的配置，从而使用`ie:https://www.google.com/`这种形式拉起IE浏览器，该办法并非通用(因为没人会想要新加一个注册表，不管他是不是programmer，虽然我做的功能是做一个通过web进行游戏内部登录，但我的测试哥哥们还是不想使用注册表的形式来做)。

说了这么多，来看下注册表实现的形式是怎样的。

IE-Protocol-Handler.reg :

	Windows Registry Editor Version 5.00

	[HKEY_CURRENT_USER\Software\Classes\ie]
	"URL Protocol"="\"\""
	@="\"URL:IE Protocol\""
	
	[HKEY_CURRENT_USER\Software\Classes\ie\DefaultIcon]
	@="\"explorer.exe,1\""
	
	[HKEY_CURRENT_USER\Software\Classes\ie\shell]
	
	[HKEY_CURRENT_USER\Software\Classes\ie\shell\open]
	
	[HKEY_CURRENT_USER\Software\Classes\ie\shell\open\command]
	@="cmd /k set myvar=%1 & call set myvar=%%myvar:ie:=%% & call \"C:\\Program Files (x86)\\Internet Explorer\\iexplore.exe\" %%myvar%% & exit /B"

保存后，你就可以在浏览器中输入ie:https://www.google.com/就可以使用IE浏览器打开了。


## 2.通过JS在IE浏览器中打开其他浏览器

这个问题是在搜索上述问题时，找到的一种解决方式，他和我预计的解决方式一点都不匹配！！！但我觉得这种方式可能会有用，所以记录下，这种方式是使用JS在网页上通过IE浏览器执行shell。

直接上代码：

	<script type="text/javascript>
		
		function execute(){ 
            var objShell = new ActiveXObject("wscript.shell"); 
            var cmd= "cmd /c start C:/\"Program Files (x86)\"/Google/Chrome/Application/chrome.exe \"https://www.google.com\"";
            objShell.Run(cmd,0,true); 
        } 
        execute();
	</script>

windows的亲儿子就是这么霸道！

最后的解决办法，还是得去适配低版本的Chrome内核，如果版本不能使用NPAPI插件，就让他下一个微端进入游戏(微端是我们自己写的带插件的浏览器)，我只是去调通了JS调用NPAPI插件的步骤在IE内核和Chrome内核下都可以拉起游戏，所以这个事情就这么解决啦！