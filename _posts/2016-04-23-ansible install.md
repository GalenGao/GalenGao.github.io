---
layout: post
title:  "centos ansible安装"
date:   2016-04-23 14:25:20 +0700
categories: [ansible]
---

# 配置epel源  

	wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo

# 安装ansible

	yum install -y ansible

# 验证安装

	ansible --version

# 配置SSH连接

    ssh-keygen -t rsa
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.120
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.121
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.122

*IP根据自己的情况去修改*  
注：如果报以下错误时，安装依赖包即可  

    错误： -bash: ssh-copy-id: command not found
    解决： yum -y install openssh-clients

# 安装ansible 2.0
## 先安装pip  

    wget https://pypi.python.org/packages/source/p/pip/pip-8.0.2.tar.gz#md5=3a73c4188f8dbad6a1e6f6d44d117eeb
    tar -zxvf pip-8.0.2.tar.gz -C /usr/local
    cd /usr/local/pip-8.0.2
    python setup.py install
    yum install gcc python-devel

## 然后安装ansible

	pip install ansible

	pip install PyCrypto==2.3