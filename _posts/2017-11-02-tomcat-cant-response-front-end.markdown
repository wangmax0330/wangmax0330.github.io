---
layout:     post
title:      "Tomcat假死,无法响应前端请求原因分析"
subtitle:   ""
date:       2017-11-02
author:     "ThreeEggs"
header-img: "img/20171014171937.jpg"
catalog: true
tags:
    - Tomcat
    - 解决方法
---

[TOC]

##现象

最近监控服务发现有台tomcat 的应用出现了无法访问的情况 ，由于已做了集群，基本没有影响线上服务的正常使用。

​        下面来简单描述该台tomcat当时具体的表现：客户端请求没有响应，查看服务器端tomcat 的java 进程存活，查看tomcat 的catalina.log ,没有发现异常，也没有error 日志.查看localhost_access.log 也没有最新的访问日志，该台tomcat 已不能提供服务。

 根据前面的假死表象，最先想到的是网络是否出现了问题，于是开始从请求的数据流程开始分析。由于业务的架构采用的是nginx + tomcat 的集群配置，一个请求上来的流向可以用下图来简单的描述。

![](https://raw.githubusercontent.com/wangmax0330/Myblog/master/image/c487bfdf-7daf-34bb-94a3-60092df97be9.jpg)



##分析

 

###网络原因

从上图可以看出，如果是网络的原因，可以从两个点进行分析:

####从前端到nginx的网络情况

​        分析nginx上的access.log ，在其中一台上可以查出当时该条请求的访问日志，也就是说可以排除这段网络的问题。

####从nginx 到tomcat 的网络情况

​        分析tomcat 的访问日志localhost_acess.log 上无法查出该条请求的访问日志。可以怀疑是否网络有问题。就该情况，从该台nginx ping 了一下tomcat server ，均为正常，没有发现问题。既然网络貌似没有问题，开始怀疑是tomcat本身的问题，在tomcat本机直接curl 调用该条请求，发现仍然没有响应。到此基本可以断定网络没有问题，tomcat 本身出现了假死的情况。

###Tomcat 原因

####tomcat jvm 内存溢出

​        分析当时的gc.log

​        基于tomcat 假死的情况，开始分析有可能的原因。造成tomcat假死有可能的情况大概有以下几种：

