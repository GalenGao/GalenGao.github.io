---
layout: post
title:  "linux下的yum使用技巧"
date:   2016-04-12 09:25:20 +0700
categories: [linux, yum]
---

经常会遇上一些linux系统允许你上外网，而一些是不允许的，这时我们可以从可以上外网的服务器上把yum下载的包拷贝过来，但是一般yum安装的包没有报错包文件，无法拷贝，为了解决这个问题，这里介绍一些小技巧。  

# 安装一般依赖包方法：  
- 如果linux系统有外网，直接yum install就可以安装，可以用yum list查看
- 如果没有外网，可以利用光盘搭建一个本地源，然后直接yum安装。

# 利用光盘配置本地源方法：  
**1、挂载光盘**

	mount /dev/cdrom /mnt 

**2、删除/etc/yum.repos.d目录所有的repo文件**  
保险起见，我们先备份一下/etc/yum.repos.d目录  

	cp -r /etc/yum.repos.d /etc/yum.reps.d.bak
	rm -rf /etc/yum.repos.d/*

**3、创建新文件dvd.repo**  

	vim /etc/yum.repos.d/dvd.repo 
    //加入以下内容： 
    [dvd] name=install dvd 
    baseurl=file:///mnt 
    enabled=1  --是否生效1是0否
    gpgcheck=0 --是否检查1检查0不检查

**4、 刷新 repos 生成缓存** 

	yum makecache

然后就可以使用yum命令安装你所需要的软件包了。如果不想使用本地yum源，需要删除掉这个/etc/yum.repos.d/dvd.repo文件，然后恢复原来的配置文件。

# 假如有两台linux，一台可以上网另外一台不能，可以利用yum在能上网的那台下到本地再传过去  
有时，我们需要下载一个rpm包，只是下载下来，拷贝给其他机器使用。前面也介绍过yum安装rpm包的时候，首先得下载这个rpm包然后再去安装。  
**所以使用yum完全可以做到只下载而不安装。**  
## 安装 yum-downloadonly 

	yum install -y yum-plugin-downloadonly.noarch 

注：如果你的CentOS是5.x版本，则需要安装yum-downloadonly.noarch这个包。  

## 下载一个rpm包而不安装

	yum install （包名） -y --downloadonly

这样虽然下载了，但是并没有保存到我们想要的目录下，它默认保存到了/var/cache/yum/后面还有好几层子目录，根据你系统的版本决定。  
在这里，我要说下，并不是所有包都可以下载，因为已经安装过的包，是不能再安装的，所以就不能下载到。  
那要是下载的话，需要使用  

	yum reinstall （包名） -y --downloadonly

## 下载到指定目录

	yum install 包名 -y --downloadonly --downloaddir=/usr/local/src

# 使用yum时出现如下错：

    another app is currently holding the yum lock;waiting for it to exit...
 
可以通过强制关掉yum进程：

	rm -f /var/run/yum.pid
 
然后就可以使用yum了。
