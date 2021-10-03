---
title: "centos如何安装软件"
tags: [linux, centos, redhat, rpm, yum, install]
---

# 背景

之前用的linux操作系统移植都是ubuntu，没有用过redhat版本的linux，最近开始想学习redhat版本的linux，就从centos开始。在安装完centos以后，第一个碰到的问题就是如何安装软件。之前在ubuntu中如何安装软件我已经写了一篇博客了，可以参考：[ubuntu下安装程序的三种方法](http://www.cnblogs.com/xwdreamer/p/3623454.html) 。下面开始将如何在centos下安装软件。

# centos安装软件的命令

[CentOS 下 rpm包与 yum 安装与卸载](http://cissco.iteye.com/blog/397163)

## 一、rpm包的安装

```text
   1.安装一个包
　　# rpm -ivh
　　2.升级一个包
　　# rpm -Uvh
　　3.移走一个包
　　# rpm -e
　　4.安装参数
　　--force 即使覆盖属于其它包的文件也强迫安装
　　--nodeps 如果该RPM包的安装依赖其它包，即使其它包没装，也强迫安装。
　　5.查询一个包是否被安装
　　# rpm -q < rpm package name>
　　6.得到被安装的包的信息
　　# rpm -qi < rpm package name>
　　7.列出该包中有哪些文件
　　# rpm -ql < rpm package name>
　　8.列出服务器上的一个文件属于哪一个RPM包
　　#rpm -qf
　　9.可综合好几个参数一起用
　　# rpm -qil < rpm package name>
　　10.列出所有被安装的rpm package
　　# rpm -qa
　　11.列出一个未被安装进系统的RPM包文件中包含有哪些文件？
　　# rpm -qilp < rpm package name
```

## 二、rpm包的卸载

```text
  rpm -qa | grep 包名
     这个命令是为了把包名相关的包都列出来     
      rpm -e 文件名
    这个命令就是你想卸载的软件，后面是包名称，最后的版本号是不用打的
   例如：
     # rpm -qa |  grep mysql
      mod_auth_mysql-2.6.1-2.2 
      php-mysql-5.3.9-3.15 
      mysql-devel-5.1.77-1.CenOS 5.2
      mysql-5.0.77-1.CenOS 5.2
      mysqlclient10-5.0.77-1.CentOS 5.2
      libdbi-dbd-mysql-0.6.5-10.CentOS 5.2
   # rpm -e mysqlclien
```

##  三、yum安装：

```text
       # yum install 包名
```

## 四、yum卸载：

```text
    # yum -y remove 包名
```

# **配置本地yum源**

**参考文献：**[CentOS yum 源的配置与使用](http://www.cnblogs.com/mchina/archive/2013/01/04/2842275.html)

## 1、挂载系统安装光盘

（挂在本地光盘可以参考：[CentOS5.5挂载本地ISO镜像](http://www.cnblogs.com/xwdreamer/p/4078465.html)）

```text
# mount /dev/cdrom /mnt/cdrom/
```

## 2、配置本地yum源

```text
# cd /etc/yum.repos.d/
# ls
```

会看到四个repo 文件

* CentOS-Base.repo 是yum 网络源的配置文件
* CentOS-Media.repo 是yum 本地源的配置文件
* ...

修改CentOS-Media.repo

```text
# cat CentOS-Media.rep
# CentOS-Media.repo
#
# This repo is used to mount the default locations for a CDROM / DVD on
#  CentOS-5.  You can use this repo and yum to install items directly off the
#  DVD ISO that we release.
#
# To use this repo, put in your DVD and use it with the other repos too:
#  yum --enablerepo=c5-media [command]
#  
# or for ONLY the media repo, do this:
#
#  yum --disablerepo=\* --enablerepo=c5-media [command]
 
[c5-media]
name=CentOS-$releasever - Media
baseurl=file:///media/CentOS/
        file:///mnt/cdrom/
        file:///media/cdrecorder/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS
```

在baseurl 中修改第2个路径为/mnt/cdrom（即为光盘挂载点）

将enabled=0改为1

## 3、禁用默认的yum 网络源

将yum 网络源配置文件改名为 CentOS-Base.repo.bak，否则会先在网络源中寻找适合的包，改名之后直接从本地源读取。

## 4、执行yum 命令

```text
# yum install postgresql
```

**关于repo 文件的格式**

所有repository 服务器设置都应该遵循如下格式：

```text
[serverid]
name=Some name for this server
baseurl=url://path/to/repository/
```

* serverid 是用于区别各个不同的repository，必须有一个独一无二的名称；
* name 是对repository 的描述，支持像$releasever $basearch这样的变量；
* baseurl 是服务器设置中最重要的部分，只有设置正确，才能从上面获取软件。它的格式是：

```text
baseurl=url://server1/path/to/repository/
　　　　 url://server2/path/to/repository/
　　　　 url://server3/path/to/repository/
```

其中url 支持的协议有 http:// ftp:// file:// 三种。baseurl 后可以跟多个url，你可以自己改为速度比较快的镜像站，但baseurl 只能有一个，也就是说不能像如下格式：

```text
baseurl=url://server1/path/to/repository/
baseurl=url://server2/path/to/repository/
baseurl=url://server3/path/to/repository/
```

其中url 指向的目录必须是这个repository header 目录的上一级，它也支持$releasever $basearch 这样的变量。  
url 之后可以加上多个选项，如gpgcheck、exclude、failovermethod 等，比如

```text
[updates-released]
name=Fedora Core $releasever - $basearch - Released Updates
baseurl=http://download.atrpms.net/mirrors/fedoracore/updates/$releasever/$basearch
　　　　 http://redhat.linux.ee/pub/fedora/linux/core/updates/$releasever/$basearch
　　　　 http://fr2.rpmfind.net/linux/fedora/core/updates/$releasever/$basearch
gpgcheck=1
exclude=gaim
failovermethod=priori
```

其中gpgcheck，exclude 的含义和\[main\] 部分相同，但只对此服务器起作用，failovermethode 有两个选项roundrobin 和priority，意思分别是有多个url可供选择时，yum 选择的次序，roundrobin 是随机选择，如果连接失败则使用下一个，依次循环，priority 则根据url 的次序从第一个开始。如果不指明，默认是roundrobin。

# References

[原文](https://www.cnblogs.com/xwdreamer/p/3813880.html)

