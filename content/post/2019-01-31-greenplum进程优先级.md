---
layout:      post
title: "Greenplum进程优先级"
subtitle:  ""
excerpt:     ""
author: "徐鹏"
date: 2019-01-31T10:20:27+08:00
description: ""
image:       ""
published: true 
tags: 
    - greenplum
    - nice
categories:  ["TIPS" ]
---
GreenPlum（以下简称GP）查询会独立启动一个进程，该进程的nice为主进程的nice+20；在我们实际生产环境下，Greenplum的进程nice为0，查询进程优先级为19（nice最大值为19）。
在GP使用独立服务器的时候是没有问题，但是其使用率较低，因此对GP集群进行了混布，混布之后发现，GP的查询性能下降。经分析为GP 查询进程的nice 为19，优先级最低。


