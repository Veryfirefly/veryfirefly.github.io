---
layout: post
title: 工作中遇到的一些问题及解决方法
date: 2019-07-06
Author: 来自veryfirefly
categories: 
tags: [工作经验,解决方法]
comments: true
---

经常会在工作遇到一些小问题，所以这篇文章是用来记录我在工作中遇到的问题和解决的办法。

## 1.Linux Bash的问题

我们经常会在**windows**平台下编写shell脚本，编写完成后会将bash脚本上传到linux上执行，赋予了对应的权限后，会出现**报错**：<font color="red">/bin/sh^m:bad interpreter:</font>

第一次遇见这个问题时，我很好奇为什么赋予了权限执行也不成功，于是我使用了"面向百度编程"，查到问题出现在windows平台下雨linux平台下的编码不一致导致的。在windows平台下我们编写的shell脚本为dos格式，而linux只能执行unix格式的脚本。这一点我们可以在vim编辑中验证。

在windows下创建一个shell脚本**simple.sh**，并编写：


    #!/bin/bash
	echo "test shell"

上传至linux机器上，我这里直接赋予了 `chmod 777 simple.sh`，执行./simple.sh，会出现<font color="red">/bin/sh^M: bad interpreter: No such file or directory</font>，我们使用vim或者vi打开simple.sh `vim simple.sh`，并输入`:set fileformat`查看编码，发现该shell脚本格式为`:set fileformat=dos`，为了能让该脚本在unix上执行，我们输入`:set fileformat=unix`，按下回车，接着输入`:wq`进行保存，在此执行脚本就成功了。


## 2.Java中 MessageDigest MD5加密不一致的问题

这个问题比较坑，前几天和接入方进行接口对接时，我编写的那个接口需要按照参数英文首字母从小到大进行排序拼接后，加上SecretKey进行MD5加密验证。但是自己测试时签名死活不通过，加密的原串一模一样，我写了一个Test类进行签名，发现签名是对的，而在tomcat下的工程加密出来的串却不一样。就这个问题折腾了半天时间。后面耐下心捋了一下，排除了tomcat传输时乱码的问题，就定位问题在MessageDigest中。

我的环境如下：

> 1. JDK1.8
> 2. Tomcat 8.5 在Connector下配置了`URIEncoding="UTF-8"`，传输的字符串都是`UTF-8`的

我直接使用的`MessageDigest`进行MD5签名，代码如下：

	private final static String[] HEXDIGITS = { "0", "1", "2", "3", 
				"4", "5", "6", "7", 
				"8", "9", "A", "B", 
				"C", "D", "E", "F" };

	
	public static String encryptMD5(String text) {
		try {
			MessageDigest mg = MessageDigest.getInstance("MD5");
			return byteArrayToString(mg.digest(text.getBytes()));
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	private static String byteArrayToString(byte[] b) {
		StringBuffer resultSb = new StringBuffer();
		for (int i = 0; i < b.length; i++) {
			resultSb.append(byteToHexString(b[i]));
		}
		return resultSb.toString();
	}

	private static String byteToHexString(byte b) {
		int n = b;
		if (n < 0) {
			n = 256 + n;
		}
		int d1 = n / 16;
		int d2 = n % 16;
		return HEXDIGITS[d1] + HEXDIGITS[d2];
	}

因为这个Utils是我的上级写的，当我接手这个项目的时候，并没有出现会用中文加密的情况，所以看了这一块代码后，我觉得 No Problem！

于是当出现问题时，我先查看了Tomcat编码，之后在检查了`HttpServletRequest`的编码，发现都是UTF-8的。咦！没问题啊，但签名怎么就不一致呢？！

![image](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1562401257890&di=4637d9f7b4c4b74c3848bec758faa7ae&imgtype=0&src=http%3A%2F%2Fb-ssl.duitang.com%2Fuploads%2Fitem%2F201608%2F05%2F20160805104608_jsMaW.thumb.224_0.jpeg)

于是我把工程中的原始字符串打出来，发现和我传入的是一样的，但在windows平台下工程加密和使用工具加密却不一致，放入外网linux机器下，签名却奇迹般的通过了！！！WTF？ 我突然意识到事情没这么简单，核对了下Utils包下的MD5加密代码后，我感觉问题就出现在`byteArrayToString(mg.digest(text.getBytes()));`下，于是我把text.getBytes()得到byte[]通过new String()得到一个字符串后，我惊奇的发现竟然乱码了！！！带着种种疑惑和上级的提示，我打开了“面向百度编程”，快速的在搜索栏中键入：JVM在Windows与Unix平台下的编码，得到了这么一个东东：[JVM默认编码](https://blog.csdn.net/qq_21033663/article/details/53022797)。该文中提到，在windows平台下JVM默认编码为GBK，而在Unix平台下默认编码为UTF-8，这也就解释了为啥签名在外网机器中签名通过的问题。

回过头我在捋了一遍代码，在 `mg.digest(text.getBytes())`中，我们并没有指定编码，而默认使用了JVM的默认编码，导致将传入的参数进行`parameter.getBytes("GBK")`，这怎么可能不乱码呢！于是我将`mg.digest(text.getBytes("UTF-8"))后，windows平台下签名通过！真的是坑啊！当然你可以使用上文中提到的方式在catalina.sh脚本中配置：JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=utf-8"来解决该问题。