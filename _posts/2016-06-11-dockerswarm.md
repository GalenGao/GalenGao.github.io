---
layout: post
title:  "docker swarm集群搭建"
date:   2016-06-11 14:25:20 +0700
categories: [docker]
---

**摘要:**  

* **swarm是docker原生的集群管理软件，与kubernetes比起来比较简单**



**1、部署**  

系统时centos7上  
关闭防火墙 systemctl stop firewalld.service   
关闭selinux vi /etc/selinux/comfig  

192.168.10.140 swarm manager  
192.168.10.141 swarm node  
192.168.10.142 swarm mode  

**2、分别在manager节点和node节点上安装docker**  

安装方式参照我的另一篇文章docker安装<http://galengao.github.io/docker/2016/06/03/mydocker-use.html>  
{% highlight ruby %}
yum update

tee /etc/yum.repos.d/docker.repo<<EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum install docker-engine
{% endhighlight %}

**3、分别在manager节点和node节点上配置docker**  

{% highlight ruby %}
vi /lib/systemd/system/docker.service
# 修改ExecStart项为如下：
ExecStart=/usr/bin/docker daemon -H 0.0.0.0:2375 -H unix:///var/run/docker.sock
# 重新载入配置，使修改生效。
systemctl daemon-reload
# 重启docker。
systemctl restart docker
{% endhighlight %}

**4、在manager节点和node节点上push swarm镜像**  

{% highlight ruby %}
[root@swarm1 ~]# docker pull swarm
Using default tag: latest
latest: Pulling from library/swarm
1e61bbec5d24: Pull complete 
8c7b2f6b74da: Pull complete 
245a8db4f1e1: Pull complete 
Digest: sha256:661f2e4c9470e7f6238cebf603bcf5700c8b948894ac9e35f2cf6f63dcda723a
Status: Downloaded newer image for swarm:latest
{% endhighlight %}

**5、创建集群token，获取全球唯一的 token，作为集群唯一标识**  

{% highlight ruby %}
# 在任何节点都可以，但是要记住该值，以后要用到
[root@swarm1 ~]# docker run --rm swarm create
eca9b4ab85feb53f8a9676c72dd01b1a
{% endhighlight %}

**6、加入集群**  

{% highlight ruby %}
# 在manager也就是节点node1
[root@swarm1 ~]# docker run -d swarm join -addr=192.168.10.140:2375 token://eca9b4ab85feb53f8a9676c72dd01b1a
109da11914295c588c6afe5f83ab731bd0d0012897c39c311de89534e2f5bc13
# node2上
[root@swarm1 ~]# docker run -d swarm join -addr=192.168.10.141:2375 token://eca9b4ab85feb53f8a9676c72dd01b1a
1da02eb6a00a8860eefe965a0aded446aebff8b502962c717dd3f494b546841a
# node3上
[root@swarm1 ~]# docker run -d swarm join -addr=192.168.10.142:2375 token://eca9b4ab85feb53f8a9676c72dd01b1a
b5483c91bff0ad21e19700af51990d631e991f9d67188c7419f147652d494972
{% endhighlight %}

**7、启动管理机**  

{% highlight ruby %}
# 在管理机上执行:
[root@swarm1 ~]# docker run -d -p 2376:2375 swarm manage token://eca9b4ab85feb53f8a9676c72dd01b1a
3073a3dd59a5782f706d6481cfd1a36e8090f21764dfec2532899450bd719456
{% endhighlight %}

**8、查看节点信息**  

{% highlight ruby %}
# 本机上查看节点信息
[root@swarm1 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
1da02eb6a00a        swarm               "/swarm join -addr=19"   27 minutes ago      Up 27 minutes       2375/tcp            sick_bose
# 查看集群所有节点信息，在任何一台机器上执行
[root@swarm1 ~]# docker run --rm swarm list token://eca9b4ab85feb53f8a9676c72dd01b1a
192.168.10.142:2375
192.168.10.141:2375
192.168.10.140:2375
# 查看集群详细信息。在任何一台机器上执行:
# 该IP地址是manager的地址
[root@swarm1 ~]# docker -H 192.168.10.140:2376 info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: swarm/1.2.3
Role: primary
Strategy: spread
Filters: health, port, containerslots, dependency, affinity, constraint
Nodes: 3
 (unknown): 192.168.10.142:2375
  └ ID: 
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ UpdatedAt: 2016-07-28T07:54:39Z
  └ ServerVersion: 
 (unknown): 192.168.10.141:2375
  └ ID: 
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ UpdatedAt: 2016-07-28T07:54:39Z
  └ ServerVersion: 
 (unknown): 192.168.10.140:2375
  └ ID: 
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ UpdatedAt: 2016-07-28T07:54:39Z
  └ ServerVersion: 
Plugins: 
 Volume: 
 Network: 
Kernel Version: 3.10.0-229.el7.x86_64
Operating System: linux
Architecture: amd64
CPUs: 0
Total Memory: 0 B
Name: 3073a3dd59a5
Docker Root Dir: 
Debug mode (client): false
Debug mode (server): false
WARNING: No kernel memory limit support
{% endhighlight %}




