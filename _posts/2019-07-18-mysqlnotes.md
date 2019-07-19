---
layout: post
title: MySQL跨库联表查询
date: 2019-07-18
Author: 来自veryfirefly
categories: 
tags: [工作经验,解决方法]
comments: true
---

已经有好几天没有写博客了，最近遇到一个问题，我们公司中的游戏日志表现在已经上报到后台表中，但因为数据量太大，导致要进行分表。我的主管所做的分表策略为：将日志表及游戏角色表独立到另一个数据库中，根据游戏服务器索引来作为分表的策略。因为日志表和角色表对后台的数据查询起着至关重要的因素，所以根据策划的需求来实现的功能，性能及其差！慢到不说，资源也占满！原本**联表查询**的逻辑因为**分库分表(两个库)**的原因只有拆分成**逻辑联表查询**(分别在两个库中查询后，在内存中进行比对)。且不说其最终数据是否一致，在游戏服日益渐增的情况下，暴露的缺陷却越来越多，原本为了强化查询的数据只能在后台而进行的游戏日志上报却成了我们这半年改了又改的还改不好的软肋！

首先，说下现在遇到的问题：

1. **logserver(就一个servlet工程)**不能停止，如果上报数据没有写库成功而造成丢失，那只有去游戏库中进行数据筛选并同步了。
2. 后台联表查询逻辑被拆分成内存联表查询，查询数据量大时会造成JVM cpu及memory资源的占用。
3. 后台内存联表查询逻辑缺陷，会查询两个库，因为数据量大造成两台物理机的mysql进程久占cpu资源导致Tomcat处理请求延迟并假死。

造成这个问题的原因是策划要求在某些查询的功能上加上一个查询角色创建时间的功能点，而该功能点就必须要查询另一个库上报过来的角色表，如果全服的话就会查询所有角色表！再查询充值表中的数据，在内存中进行数据比对！数据量小一点还好说，但是游戏运行了快3个月，角色数据和充值数据数据量特别大，所以如果用以上逻辑功能进行查询会造成功能查询慢、无反应、并且会造成100%的cpu占用、Tomcat响应慢甚至假死、并且mysql还在查询，极有可能造成物理机直接宕机的问题。

## 1.使用JMX监控Tomcat

前几天策划突然反映我们的后台刷新慢、登不进去的情况时，我先开始想用JMX查看Tomcat的cpu、memory的占用率来找出问题，然后在catalina.sh中加入了开启JMX的参数：

	CATALINA_OPTS="-Xms1024m -Xmx6144m -XX:+HeapDumpOnOutOfMemoryError -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
					-Dspring.profiles.active=production -Xloggc:/data/logs/gc-`date +"%Y-%m-%d_%H%M%S"`.log 
					-XX:MaxPermSize=1024M -Dcom.sun.management.jmxremote=true 
					-Djava.rmi.server.hostname=192.168.1.3 -Dcom.sun.management.jmxremote.port=20954 
					-Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.managementote.ssl=false 
					-Dcom.sun.management.jmxremote.authenticate=false 
					-XX:+UnlockCommercialFeatures -XX:+FlightRecorder"

> 上面的IP是自己测试的IP，如果要使用请换成自己的出口IP

满怀信心的开启了JMX，准备等等问题自动复现的执行了./start.sh脚本开启Tomcat，去到logs下看catalina.out查看Tomcat启动情况，结果发现没启起来！错误如下：

	Unrecognized VM option 'UnlockCommercialFeatures'
	Error: Could not create the Java Virtual Machine.
	Error: A fatal exception has occurred. Program will exit.

WTF？然后我把 `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder`去掉后，启起来了，但是JMC、JConsole、JVisualVM打死都连不上，于是我只有去Google查询问题，然后终于找到JMX无法启动的问题所在，因为我那台机器上的JDK是OpenJDK！！！

OpenJDK与JDK的差异：

1. **授权协议的不同**：OpenJDK采用GPL V2协议放出，而JDK则采用JRL（JavaResearch License，Java研究授权协议）放出。两者协议虽然都是开放源代码的，但是在使用上的不同在于GPL V2允许在商业上使用，而JRL只允许个人研究使用。 
2. **OpenJDK不包含Deployment（部署）功能**：部署的功能包括：Browser Plugin、Java Web Start、以及Java控制面板，这些功能在OpenJDK中是找不到的。
3. **OpenJDK源代码不完整**：这个很容易想到，在采用GPL协议的OpenJDK中，Sun JDK的一部分源代码因为产权的问题无法开放OpenJDK使用，<u>其中最主要的部份就是JMX中的可选元件SNMP部份的代码。因此这些不能开放的源代码将它作成plugin，以供OpenJDK编译时使用，你也可以选择不要使用plugin。</u>而Icedtea则为这些不完整的部分开发了相同功能的源代码(OpenJDK6)，促使OpenJDK更加完整。
4. **部分源代码用开源代码替换**：由于产权的问题，很多产权不是SUN的源代码被替换成一些功能相同的开源代码，比如说字体栅格化引擎，使用Free Type代替。 
5. **OpenJDK只包含最精简的JDK**： OpenJDK不包含其他的软件包，比如Rhino Java DB JAXP……，并且可以分离的软件包也都是尽量的分离，但是这大多数都是自由软件，你可以自己下载加入。 

OK！查到这个文档后，我直接放弃了JMX！([JDK-8173135](https://bugs.openjdk.java.net/browse/JDK-8173135))

## 2.使用Arthas

这个我就很快的说一下吧，我当时直接想到Arthas，但是我并没有完全使用过这个工具，对其中很多的指令都不怎么清晰，所以我也只是查看的dashboard，我发现我监控的Tomcat占用资源并不高，所以也就排除Tomcat的问题了。

开启Arthas

![img](https://veryfirefly.github.io/images/arthas-open.png)

进入dashboard

![img](https://veryfirefly.github.io/images/arthas-dashboard.png)

可见Tomcat占用CPU及内存也并不高，问题不是Tomcat直接产生的。

## 3.使用top命令

JMX不能使用，Arthas不能定位问题，那么我只有祭出top大法了！

于是直接键入top，果然发现了问题！mysqld进程的cpu占用率从4.2%突然飙升到78.4%，OMG！！！

![img](https://veryfirefly.github.io/images/mysql-cpu.png)

终于定位到问题所在了，发现就是付费逻辑数据量激增，导致一直在查询mysql数据库，共有15万条sql在查询。

## 4.MySQL FEDERATED Engine

既然只有一张charge表在后台，那么能不能做一个跨库联表查询呢？就像这样：

	select 
		condition 
	from 
		192.168.0.1@database.table a 
	inner join 
		192.168.0.2@database.table b
	on
		a.id = b.id

想法很天真~不过MySQL确实有另一种实现方式来实现该功能，那就是.......**FEDERATED引擎**！！！

FEDERATED引擎可以帮助我们在遇到需要操作其他数据库的部分表时，进行数据表映射关联。

例如：当A数据库中有一张a数据表，而B数据库中需要操作A数据库中的a表时，但又不能与A数据库进行连接关联，那么这时候可以在B数据库中创建一张b数据表，其结构和A库中的a表相同，此时B'b其实只是一个链接，它只存有A'a的数据表结构，而B'b表的物理数据都只存在于A'a表中，通俗的解释B'b表就只是一个软链接或者是快捷方式。当查询时由MySQL内部进行远程请求。B'b表具有A'a表的Select、Update、Insert、Delete功能。当在B'b表中增删改数据时，A'a表也会做相应的更改。

**FEDERATED引擎注意事项**：

- 本地的表结构必须与远程的表结构完全一致。
- 远程数据库目前仅限于MySQL。
- 不支持事务。
- **不支持表结构的修改**。

> 开启FEDERATED引擎

我们先查询MySQL引擎：

	show engines;

![img](https://veryfirefly.github.io/images/show-engines.png)

发现没有FEDERATED引擎，于是我们安装它。

	install plugin federated soname 'ha_federated.so';

![img](https://veryfirefly.github.io/images/install-federated.png)

再次查看MySQL引擎，发现有FEDERATED引擎了，但support为NO

![img](https://veryfirefly.github.io/images/show-federated.png)

然后在my.cnf中配置federated开启该引擎。

![img](https://veryfirefly.github.io/images/config-mycnf.png)


重启MySQL服务。

![img](https://veryfirefly.github.io/images/restart-mysqld.png)

再次查看MySQL引擎，发现support已经为yes了。

![img](https://veryfirefly.github.io/images/reshow-engines.png)


> 关联远程数据表

	CREATE TABLE `charge` (
  		`id` bigint(20) NOT NULL AUTO_INCREMENT,
  		`username` varchar(20) NOT NULL,
  		`password` varchar(32) NOT NULL,
  		`createtime` datetime NOT NULL,
  		PRIMARY KEY (`id`),
  		UNIQUE KEY `username` (`username`),
  		KEY `createtime` (`createtime`),
	) ENGINE=FEDERATED DEFAULT CHARSET=latin1 CONNECTION='mysql://root:mysqladmin@192.168.1.3:3306/shop/user';

表结构与远程数据库相同，在后面加上CONNECTION='mysql://root(<font color='red'>用户名</font>):mysqladmin(<font color='red'>密码</font>)@192.168.1.3(<font color='red'>地址</font>):3306(<font color='red'>端口</font>)/shop(<font color='red'>数据库</font>)/user(<font color='red'>表名</font>)'

创建完成后，就可以进行跨库联表操作啦！


