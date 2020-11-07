---
title: "用Hugo、Nginx、Docker搭建静态博客(一) 发布"
categories: ["Hugo"]
tags: ["hugo","Docker", "Nginx","静态博客"]
date: 2020-11-01T09:01:07+08:00
draft: false
toc: true
---

## 服务器

在第一篇安装与介绍文章中，相信大家已经能够在本地把博客跑起来。但是我们如果想要把博客分享到互联网上，必须有一个托管的云服务器。或者在家里自己折腾端口映射，把自己的博客地址暴露到公网。

这里笔者购买了阿里云的服务器，操作系统为Li nux CentOS 8.2, 内核版本为4.18。 详细服务器的版本信息如下： 

```bash 
[jack@ ~]$ cat /etc/centos-release
CentOS Linux release 8.2.2004 (Core) 
[jack@ ~]$ 
[jack@ ~]$ uname -a
Linux 4.18.0-193.14.2.el8_2.x86_64 #1 SMP Sun Jul 26 03:54:29 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
[jack@ ~]$

```

## 安装必备 

### git 
怎么安装git,请大家参考git官网，网上资料很多，这里就不在赘述。 例如这里： [如何在centos 8上安装git ](https://blog.csdn.net/cukw6666/article/details/107983303)

### docker 
网上安装教程较多，这里省略。

### hugo
安装请参考第一篇的内容。







