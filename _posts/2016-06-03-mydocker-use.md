---
layout: post
title:  "我的docker全套流程例子"
date:   2016-06-03 14:25:20 +0700
categories: [docker]
---

**摘要:**  

* **下文是自己从搭建docker到docker里安装mysql到push的一遍大概过程**
* **docker的安装直接引用官方文档，英文比较简单，所以没有多加翻译**


目前红帽RHEL系统下面安装docker可以有两种方式：一种是使用curl获得docker的安装脚本进行安装，还有一种是使用yum包管理器来安装docker。

# Install on CentOS

Docker runs on CentOS 7.X. An installation on other binary compatible EL7 distributions such as Scientific Linux might succeed, but Docker does not test or support Docker on these distributions.  
This page instructs you to install using Docker-managed release packages and installation mechanisms. Using these packages ensures you get the latest release of Docker. If you wish to install using CentOS-managed packages, consult your CentOS documentation.  

## Prerequisites

Docker requires a 64-bit installation regardless of your CentOS version. Also, your kernel must be 3.10 at minimum, which CentOS 7 runs.  
To check your current kernel version, open a terminal and use uname -r to display your kernel version:  

{% highlight ruby %}
$ uname -r
3.10.0-229.el7.x86_64  
{% endhighlight %}

Finally, it is recommended that you fully update your system. Please keep in mind that your system should be fully patched to fix any potential kernel bugs. Any reported kernel bugs may have already been fixed on the latest kernel packages.  

## Install  

There are two ways to install Docker Engine. You can install using the yum package manager. Or you can use curl with the get.docker.com site. This second method runs an installation script which also installs via the yum package manager.  

### Install with yum  

Log into your machine as a user with sudo or root privileges.  

* **Make sure your existing yum packages are up-to-date.**   

{% highlight ruby %}
$ yum update
{% endhighlight %}

* **Add the yum repo.**  

{% highlight ruby %}
$ tee /etc/yum.repos.d/docker.repo <<EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
{% endhighlight %}

* **Install the Docker package.**  

{% highlight ruby %}
$ sudo yum install docker-engine
{% endhighlight %}

* **Start the Docker daemon.**  

{% highlight ruby %}
$ sudo service docker start
{% endhighlight %}

* **Verify docker is installed correctly by running a test image in a container.**  

{% highlight ruby %}
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
    latest: Pulling from hello-world
    a8219747be10: Pull complete
    91c95931e552: Already exists
    hello-world:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
    Digest: sha256:aa03e5d0d5553b4c3473e89c8619cf79df368babd1.7.1cf5daeb82aab55838d
    Status: Downloaded newer image for hello-world:latest
    Hello from Docker.
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
     1. The Docker client contacted the Docker daemon.
     2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
            (Assuming it was not already locally available.)
     3. The Docker daemon created a new container from that image which runs the
            executable that produces the output you are currently reading.
     4. The Docker daemon streamed that output to the Docker client, which sent it
            to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
     $ docker run -it ubuntu bash

    For more examples and ideas, visit:
     http://docs.docker.com/userguide/
{% endhighlight %}

### Install with the script

Log into your machine as a user with sudo or root privileges.  

* **Make sure your existing yum packages are up-to-date.**  

{% highlight ruby %}  
$ sudo yum update
{% endhighlight %}

* **Run the Docker installation script.**  

{% highlight ruby %} 
$ curl -fsSL https://get.docker.com/ | sh
# This script adds the docker.repo repository and installs Docker.
{% endhighlight %}

* **Start the Docker daemon.**  

{% highlight ruby %}
$ sudo service docker start
# Verify docker is installed correctly by running a test image in a container.
$ sudo docker run hello-world
{% endhighlight %}

## Create a docker group

The docker daemon binds to a Unix socket instead of a TCP port. By default that Unix socket is owned by the user root and other users can access it with sudo. For this reason, docker daemon always runs as the root user.  
To avoid having to use sudo when you use the docker command, create a Unix group called docker and add users to it. When the docker daemon starts, it makes the ownership of the Unix socket read/writable by the docker group.  
> Warning: The docker group is equivalent to the root user; For details on how this impacts security in your system, see Docker Daemon Attack Surface for details.  

To create the docker group and add your user:  
* Log into Centos as a user with sudo privileges.
* Create the docker group.  

`groupadd docker`  

* Add your user to docker group.  

`usermod -aG docker your_username`

* Log out and log back in.  
This ensures your user is running with the correct permissions.  

* Verify your work by running docker without sudo.  

`$ docker run hello-world`  

### Start the docker daemon at boot  

* To ensure Docker starts when you boot your system, do the following:  

`$ sudo chkconfig docker on`  

If you need to add an HTTP Proxy, set a different directory or partition for the Docker runtime files, or make other customizations, read our Systemd article to learn how tocustomize your Systemd Docker daemon options.  

## Uninstall

You can uninstall the Docker software with yum.  

* List the package you have installed.  

{% highlight ruby %}
$ yum list installed | grep docker
yum list installed | grep docker
docker-engine.x86_64   1.7.1-1.el7 @/docker-engine-1.7.1-1.el7.x86_64.rpm
{% endhighlight %}

* Remove the package.  

{% highlight ruby %}
$ sudo yum -y remove docker-engine.x86_64
{% endhighlight %}

This command does not remove images, containers, volumes, or user-created configuration files on your host.  

* To delete all images, containers, and volumes, run the following command:  

{% highlight ruby %}
$ rm -rf /var/lib/docker
{% endhighlight %}

Locate and delete any user-created configuration files.


# docker的使用

## 下载启动镜像安装应用

* 查看已有的镜像  

{% highlight ruby %}
docker images
{% endhighlight %}

* 搜索镜像，假如我们要下载centos6.6，这里只列出部分搜索结果来   

{% highlight ruby %}
[root@galen yum.repos.d]# docker search centos:6.6
NAME                           DESCRIPTION         STARS     OFFICIAL   AUTOMATED
                                                             
anarh/centos6.6                 Docker image for centos6.6                      0                    [OK]
eliezio/centos6.6-devtoolset2-gtest    Docker image based on Centos 6.6 suitable ...   0             [OK]
chrisgeorge/centos6.6-py2.6         CentOS 6.6 with Python 2.6                      0                [OK]
mystique/hadoopbase              Hadoop base image - Centos 6.6 Updated to ...   0                   [OK]
incu6us/centos6.6-with-nginx        Wav server for FreeCall                         0                [OK]
{% endhighlight %}

* pull镜像，上面搜索到很多，我们选择一个下载  

{% highlight ruby %}
[root@galen yum.repos.d]# docker pull anarh/centos6.6
Using default tag: latest
latest: Pulling from anarh/centos6.6
a3ed95caeb02: Pull complete 
75fcb2d165e6: Pull complete 
Digest: sha256:c2e5b2ff2b471ce747952c3887595f34410b390c9e36ba0823fbbc8b9a54c637
Status: Downloaded newer image for anarh/centos6.6:latest
{% endhighlight %}

* 这时在查看镜像发现就有了  

{% highlight ruby %}
[root@galen yum.repos.d]# docker images
REPOSITORY       TAG            IMAGE ID            CREATED            SIZE
anarh/centos6.6    latest          eeb98e74a7bd        10 months ago       202.6 MB
{% endhighlight %}

* 运行该镜像

{% highlight ruby %}
[root@galen yum.repos.d]# docker run -t -i anarh/centos6.6 /bin/bash
[root@14aa06d651c6 /]# 
{% endhighlight %}

* 然后就可以在里面像正常linux一样安装自己的软件了，这里我安装mysql，步骤省略  

## 提交容器

我们在里面编译安装完mysql后，我们就提交生成镜像，以后就可以直接启动引用该镜像了  

* `exit`后通过`docker ps -a`找到该容器names对应的容器ID  
* 运行`commit`命令来提交  

`docker commit -m "install mysql5.6.28 from centos6.6" -a "galen" 14aa06d651c6 galen/centos6.6-mysql5.6`

> 其中，-m参数用来来指定提交的说明信息；-a可以指定用户信息的；14aa06d651c6代表你的容器的id；galen/centos6.6-mysql5.6指定目标镜像的用户名、仓库名和 tag 信息。创建成功后会返回这个镜像的 ID 信息。注意的是，你一定要将galen改为你自己的用户名。因为下文还会用到此用户名!  

{% highlight ruby %}
[root@galen ~]# docker commit -m "install mysql5.6.28 from centos6.6" -a "galen" 14aa06d651c6 galen/centos6.6-mysql5.6
sha256:8fa5dde39e421b9ade3dae485f9276bd9c5b64ef5f38995afcdd3a3fb6df40f7
{% endhighlight %}

* 这是我们再次使用docker images命令就会发现此时多出了一个我们刚刚创建的镜像，此时我们就可以docker run加上你的参数运行改容器了:  

{% highlight ruby %}
[root@galen ~]# docker images
REPOSITORY               TAG              IMAGE ID            CREATED             SIZE
galen/centos6.6-mysql5.6   latest             8fa5dde39e42        57 seconds ago      3.654 GB
hello-world             latest             c54a2cc56cbb        9 days ago          1.848 kB
anarh/centos6.6           latest             eeb98e74a7bd        10 months ago       202.6 MB
{% endhighlight %}


## 存储镜像

* 登录docker hub  

我们刚刚已经创建了自己的第一个镜像，尽管它很简单，但这已经非常棒了，现在，我们希望它能够被更多的人使用到，此时，我们就需要将这个镜像上传到镜像仓库，Docker的官方Docker Hub应该是目前最大的Docker镜像中心，所以，我们就将我们的镜像上传到Docker Hub。  
首先，我们需要成为Docker Hub的用户，前往https://hub.docker.com/进行注册。需要注意的是，为了方便下面的操作，你需要将你的用户名设为和我刚刚在上文提到的自定义用户名相同，例如我的刚刚将镜像的名字命名为是galen/centos6.6-mysql5.6,所以我的用户名为galen、注册完成后记住用户名、密码、邮箱。  
login默认是用来登陆Docker Hub的，因此，输入如下命令来尝试登陆Docker Hub:  
`docker login`  
此时，就会输出交互，让我们输入Username、Password、Email,成功输入我们刚才注册的信息后就会返回Login Success提示:  

{% highlight ruby %}
[root@galen ~]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: galen
Password: 
Login Succeeded
{% endhighlight %}

注意：要把防火墙关闭，否则登不上  

* 登录后运行命令`docker push galen/centos6.6-mysql5.6`，把镜像push到docker hub上

{% highlight ruby %}
[root@galen ~]# docker push galen/centos6.6-mysql5.6
The push refers to a repository [docker.io/galen/centos6.6-mysql5.6]
57b8f026c3f7: Waiting 
5f70bf18a086: Image already pushed, skipping 
2a45600af30a: Pushing [====================>                              ] 83.93 MB/202.6 MB
{% endhighlight %}

* 上传完后你登录docker hub你的账号后，在里面就能看到你上传的东西了

## 创建私有仓库



> 注意：如果退出去了当前容器，比如`exit`，在重新执行上上述命令`docker run -t -i anarh/centos6.6 /bin/bash`是不行的，重新执行上述命令后实际是新启动一个容器，这时是没有你原来的操作过程的的，比如你上面安装的mysql。  
如何找到你上述那个容器呢？  
1、退出来后执行下面命令会把所有容器列出来
docker ps -a  

> 2、如果当初启动的时候没有指定一个可认识的name，如docker run -t -i --name xxx anarh/centos6.6 /bin/bash 只有一个一个的查看日志，看你的安装步骤了     
docker logs 容器id  
[root@galen ~]# docker logs 6409d2c066b4  
[root@6409d2c066b4 /]#   
[root@6409d2c066b4 /]# netstat | grep 22  
[root@6409d2c066b4 /]# netstat  
Active Internet connections (w/o servers)  
Proto Recv-Q Send-Q Local Address               Foreign Address             State        
Active UNIX domain sockets (w/o servers)  
Proto RefCnt Flags       Type       State         I-Node Path  
[root@6409d2c066b4 /]# yum install wget  

> 3、找到后立即把那个名字重名以后，方便以后好找。原来你没有指定name，它会自己随机一个，如上图的name列  
docker rename 旧名字 新名字  
docker rename drunk_goldberg mysql  

> 4、如果要重新加新的参数启动该容器，比如加上-p端口映射，这时只能，先提交成镜像，然后重新启动，详见其它步骤


