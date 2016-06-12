---
layout: post
title:  "Centos6 搭建Github Pages + Jekyll + Markdown"
date:   2016-04-11 15:25:20 +0700
categories: [github, jekyll]
---

查看了很多安装文档，每个人说的思路不太一样，所以安装也遇上一定的困难。我总结了几个人的文档，从安装基础软件开始安装，  
以下便是我的文档，以供参考。  

# 引言

查看了很多安装文档，每个人说的思路不太一样，所以安装也遇上一定的困难。我总结了几个人的文档，从安装基础软件开始安装，  
以下便是我的文档，以供参考。 
 
# 安装rvm  
rvm是用于安装ruby的，以下命令有时会解析不到那个地址，多尝试几次。 
 
	curl -L https://get.rvm.io | bash -s stable --ruby
	source /usr/local/rvm/scripts/rvm

执行`curl -L https://get.rvm.io | bash -s stable --ruby`会报如下错误：  
![alt text](/static/img/myimg/rvm1.png)  
**解决办法**：  
执行上面圈起来的语句，然后重新执行即可  
![solution](/static/img/myimg/rvm2.png)  
**然后执行** 

	curl -L https://get.rvm.io | bash -s stable --ruby

**使其生效**

	source /usr/local/rvm/scripts/rvm

**`rvm -v` 表示安装成功**  
![look](/static/img/myimg/rvm3.png) 

#安装ruby

这里安装2.3.0版本的，因为后面的jekyll要2.0以上的  

	rvm install 2.3.0
	rvm use 2.3.0

![ruby](/static/img/myimg/ruby1.png) 

#安装git

	yum -y git

![ruby](/static/img/myimg/git1.png)

#生成SSH KEY

	ssh-keygen -t rsa

回车，输入两次密码:  
![ssh](/static/img/myimg/ssh1.png)

#把SSH KEY添加到GITHUB里

	cat /root/.ssh/id_rsa.pub

![ssh](/static/img/myimg/ssh2.png)
把上面的内容复制到github里
=======================  
登录<https://github.com/> 
 
**github页面右上角有一个[setting]设置** 如下:
![ssh](/static/img/myimg/ssh3.png)

**进去后点SSH那个选项**
![ssh](/static/img/myimg/ssh4.png)

**然后新建ssh**
![ssh](/static/img/myimg/ssh5.png)

**最后把上面的内容复制进去，title可以空起**
![ssh](/static/img/myimg/ssh6.png)

**然后Add SSH key提交后，会让你再输一把你的github账号即可。  
测试一下，输入ssh -T git@github.com，出现如下信息即可。箭头那输入密码**
![ssh](/static/img/myimg/ssh7.png)

#设置你的账号信息

现在你已经可以通过SSH链接到GitHub了，还有一些个人信息需要完善的。  
Git会根据用户的名字和邮箱来记录提交。GitHub也是用这些信息来做权限的处理，输入下面的代码进行个人信息的设置，把名称和邮箱替换成你
自己的，名字必须是你的真名，而不是GitHub的昵称。  
    git config --global user.name "你的名字"
    git config --global user.email "your_email@youremail.com"


#使用GitHub Pages建立博客

**new repository**    
![github](/static/img/myimg/github1.png)

**起个repository名，然后创建**  
![github](/static/img/myimg/github2.png)

**这时出现如下消息**  
![github](/static/img/myimg/github3.png)

**点击上面的setting后如下**  
![github](/static/img/myimg/github4.png)

**再点击上面的Launch automatic page generator**  
![github](/static/img/myimg/github6.png)

**这时上面都不用管，继续点击上面的continue to layouts**  
![github](/static/img/myimg/github5.png)

**选中一个主题后点击上面的Publish page即可，会出现一个地址。**  
![github](/static/img/myimg/github7.png)

**复制上面的地址http://galengao.github.io/myblog在新窗口中打开找到SSH后的连接**  
![github](/static/img/myimg/github8.png)

**然后复制那个连接，在本地克隆**  
![github](/static/img/myimg/github9.png)

#安装jekyll

	gem install jekyll

如果要换镜像，如下：  

	gem sources --remove https://rubygems.org/
	gem sources -a https://ruby.taobao.org/
	gem sources -l、

如果安装jekyll出现如下错误时：    
![jekyll](/static/img/myimg/jekyll1.png)

解决升级gem，移除国外镜像，用国内的。  

	rvm rubygems latest
	gem sources --remove https://rubygems.org/
	gem sources -a https://ruby.taobao.org/
	gem sources -l

![jekyll](/static/img/myimg/jekyll2.png)

#启动项目查看情况

进入我克隆的目录  
`cd /root/GalenGao.github.io ` 
启动（注：如果不指定IP和端口，默认启动是127.0.0.1:4000，这时远程访问不了，可用jekyll serve --help） 

	jekyll serve -H192.168.10.145

![project](/static/img/myimg/project1.png)
这时浏览器会看到如下信息：
![project](/static/img/myimg/project2.png)

#替换jekyll模板

下载地址<http://jekyllthemes.org/>  
到网站去任意下载一个模板，然后上传到本地服务器。如我下载的  

	unzip unifreak.github.io-master.zip 
	cd unifreak.github.io-master.zip

把里面的内容全部拷入到我上面的项目下  

	cp -rf * ../GalenGao.github.io/

#你还需要如下命令推送到 github.

	git add .
	git commit -m "first post"
	git push origin master

#makdown工具
##markdownpad下载http://www.markdownpad.com/download.html
###注册码
邮箱：
Soar360@live.com
授权秘钥：
GBPduHjWfJU1mZqcPM3BikjYKF6xKhlKIys3i1MU2eJHqWGImDHzWdD6xhMNLGVpbP2M5SN6bnxn2kSE8qHqNY5QaaRxmO3YSMHxlv2EYpjdwLcPwfeTG7kUdnhKE0vVy4RidP6Y2wZ0q74f47fzsZo45JE2hfQBFi2O9Jldjp1mW8HUpTtLA2a5/sQytXJUQl/QKO0jUQY4pa5CCx20sV1ClOTZtAGngSOJtIOFXK599sBr5aIEFyH0K7H4BoNMiiDMnxt1rD8Vb/ikJdhGMMQr0R4B+L3nWU97eaVPTRKfWGDE8/eAgKzpGwrQQoDh+nzX1xoVQ8NAuH+s4UcSeQ==