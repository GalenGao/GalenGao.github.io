---
layout: post
title:  "CENTOS6.6上搭建单实例ORACLE12C"
date:   2016-05-26 14:25:20 +0700
categories: [oracle]
---

**摘要:**  

* **自己在centos6.6上搭建的单实例oracle12c**
* **由于搭建过程有些不好写，所以图片偏多**
* ***由于截图不规则导致排版有点乱，已经安装过来了，有些截图不能回头截图了，见谅*



* oracle软件与linux 认证版本  

![install ora](/static/img/inora/ora12c1.png)
![install ora](/static/img/inora/ora12c2.png)

* 检查硬件要求(Check Hardware Requirements)    
 **Check CPU**  
{% highlight ruby %}
grep "model name" /proc/cpuinfo
cat /proc/cpuinfo | grep "processor" | wc -l
cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
{% endhighlight %}

![install ora](/static/img/inora/ora12c3.png)
   
 **Check Memory**  

{% highlight ruby %} 
grep MemTotal /proc/meminfo
grep SwapTotal /proc/meminfo
free -g
{% endhighlight %}
![install ora](/static/img/inora/ora12c4.png)

Oracle 12c 对系统内存的最低要求为1G，推荐2G或更大的内存  
Oracle对交换分区（Swap Space）的推荐设置如下  
![install ora](/static/img/inora/ora12c5.png)
 
 **Check Disk Capacity**  
{% highlight ruby %}
df -h
{% endhighlight %}
![install ora](/static/img/inora/ora12c6.png)

> Oracle 12c 企业版的需要6.4G大小的磁盘空间，标准版需要6.1G大小的磁盘空间。/tmp 需要至少1G的大小  

* 检查软件要求(Checking the Software Requirements)  
    **操作系统版本检测**   

Oracle 12 c 只支持64位的Linux系统。不支持32Linux平台，这也许是以后的趋势了。Operating System Requirements for x86-64 Linux Platforms。   
Oracle 的官方文档明确列出了支持下面三个Linux版本:    
Supported Oracle Linux 6 and Red Hat Enterprise Linux 6 Distributions for x86-64  
Supported Oracle Linux 5 and Red Hat Enterprise Linux 5 Distributions for x86-64  
Supported SUSE Distributions for x86-64    

{% highlight ruby %}
uname -m
uname -r
more /etc/redhat-release
uname -a
lsb_release -id
{% endhighlight %}
![install ora](/static/img/inora/ora12c7.png)

 **依赖包检查**  

Packages for Oracle Linux 6 and Red Hat Enterprise Linux 6  
{% highlight ruby %}
The following packages (or later versions) must be installed:
binutils-2.20.51.0.2-5.11.el6 (x86_64)
compat-libcap1-1.10-1 (x86_64)
compat-libstdc++-33-3.2.3-69.el6 (x86_64)
compat-libstdc++-33-3.2.3-69.el6 (i686)
gcc-4.4.4-13.el6 (x86_64)
gcc-c++-4.4.4-13.el6 (x86_64)
glibc-2.12-1.7.el6 (i686)
glibc-2.12-1.7.el6 (x86_64)
glibc-devel-2.12-1.7.el6 (x86_64)
glibc-devel-2.12-1.7.el6 (i686)
ksh
libgcc-4.4.4-13.el6 (i686)
libgcc-4.4.4-13.el6 (x86_64)
libstdc++-4.4.4-13.el6 (x86_64)
libstdc++-4.4.4-13.el6 (i686)
libstdc++-devel-4.4.4-13.el6 (x86_64)
libstdc++-devel-4.4.4-13.el6 (i686)
libaio-0.3.107-10.el6 (x86_64)
libaio-0.3.107-10.el6 (i686)
libaio-devel-0.3.107-10.el6 (x86_64)
libaio-devel-0.3.107-10.el6 (i686)
libXext-1.1 (x86_64)
libXext-1.1 (i686)
libXtst-1.0.99.2 (x86_64)
libXtst-1.0.99.2 (i686)
libX11-1.3 (x86_64)
libX11-1.3 (i686)
libXau-1.0.5 (x86_64)
libXau-1.0.5 (i686)
libxcb-1.5 (x86_64)
libxcb-1.5 (i686)
libXi-1.3 (x86_64)
libXi-1.3 (i686)
make-3.81-19.el6
sysstat-9.0.4-11.el6 (x86_64)
{% endhighlight %}

{% highlight ruby %}
# 检查包
rpm -q binutils compat-libcap1 compat-libstdc++ gcc gcc-c++ glibc glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel libXext libXtst libX11 libXau libXi make sysstat
{% endhighlight %}
![install ora](/static/img/inora/ora12c8.png)

还有7个没安装  
{% highlight ruby %}
yum install compat-libcap1 compat-libstdc++ gcc gcc-c++  ksh libaio-devel libstdc++-devel
{% endhighlight %}
如yum没有的就从本地光盘或者下载来安装  

* service iptables stop  

{% highlight ruby %}
chkconfig iptables off
{% endhighlight %}

* vi /etc/selinux/config  

修改  
SELINUX=disabled  
![install ora](/static/img/inora/ora12c9.png)

* vi etc/hosts

增加  
192.168.1.140 dgp  
![install ora](/static/img/inora/ora12c10.png)

* vi /etc/security/limits.conf  
{% highlight ruby %}
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 32768

###若RAC，需要增加如下
grid soft nproc 2047
grid hard nproc 16384
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 32768
{% endhighlight %}

![install ora](/static/img/inora/ora12c11.png)

* vi /etc/sysctl.conf  

{% highlight ruby %}
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4294967295
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
{% endhighlight %}

![install ora](/static/img/inora/ora12c12.png)

> 注：因为kernel.shmall和kernel.shmmax 系统里已经有比它大的值了，所以把这两个参数注释  

sysctl -p  生效  
![install ora](/static/img/inora/ora12c13.png)

* vi /etc/profile  

{% highlight ruby %}
if [ $USER = "oracle" ]; then 
if [ $SHELL = "/bin/ksh" ]; then 
ulimit -p 16384 
ulimit -n 65536 
else 
ulimit -u 16384 -n 65536 
fi
fi
{% endhighlight %}

篇幅太长，截取部分  
![install ora](/static/img/inora/ora12c14.png)

* vi /etc/pam.d/login  

{% highlight ruby %}
session required /lib/security/pam_limits.so 
session required pam_limits.so
{% endhighlight %}

![install ora](/static/img/inora/ora12c15.png)


* 上传oracle12c软件并解压  

{% highlight ruby %}
mkdir /u01
{% endhighlight %}
![install ora](/static/img/inora/ora12c16.png)


* 创建oracle用户和用户组  

{% highlight ruby %}
groupadd dba
groupadd oinstall
useradd -g oinstall -G dba oracle
passwd oracle
chown -R oracle:oinstall /u01
{% endhighlight %}


* 启动xmanager图形界面安装  
 **如果没图形界面，需要先安装**    

{% highlight ruby %}
yum groupinstall "X Window System"
yum -y groupinstall Desktop 
runlevel
vi /etc/inittab
id:5:initdefault:
{% endhighlight %}

先启动Xmanager - Passive  
然后启动 Xstart  
输入主机地址，协议用SSH，然后弹出输入密码  

![install ora](/static/img/inora/ora12c17.png)

进入securecrt 切换到oracle用户  
su - oracle  
输入export DISPLAY=192.168.10.20：0.0 ##本地ip  

* 开始安装oracle12c  

{% highlight ruby %}
cd database
./runInstaller
{% endhighlight %}
![install ora](/static/img/inora/ora12c18.png)

* 不需要支持，弹出提示时点yes  
![install ora](/static/img/inora/ora12c19.png)


* 先仅安装数据库软件  

![install ora](/static/img/inora/ora12c20.png)


* 这里安装单实例的  
![install ora](/static/img/inora/ora12c21.png)

* 选择默认英语  
![install ora](/static/img/inora/ora12c22.png)

* 默认企业版  
![install ora](/static/img/inora/ora12c23.png)

* 软件安装路径  
![install ora](/static/img/inora/ora12c24.png)

如果空间不够，新增一块硬盘，然后格式化并挂载上去  
fdisk -l  
fdisk /dev/sdb  
![install ora](/static/img/inora/ora12c25.png)

![install ora](/static/img/inora/ora12c26.png)


* 默认  
![install ora](/static/img/inora/ora12c27.png)
 
* 默认  
![install ora](/static/img/inora/ora12c28.png)

* 安装条件检查，前面准备好一般没什么问题；  
![install ora](/static/img/inora/ora12c29.png)

* 然后开始安装  
![install ora](/static/img/inora/ora12c30.png)
![install ora](/static/img/inora/ora12c31.png)


* 配置环境变量  
![install ora](/static/img/inora/ora12c32.png)

source .bash_profile  

* dbca创建实例  
![install ora](/static/img/inora/ora12c33.png)

* 设置全局库名及密码下一步  
![install ora](/static/img/inora/ora12c34.png)

* 先决条件检查，这里空间不够先忽略  
![install ora](/static/img/inora/ora12c35.png)

* 前面设置的预览  
![install ora](/static/img/inora/ora12c36.png)

* 点结束开始安装  
![install ora](/static/img/inora/ora12c37.png)

* 检查  
至此就安装结束了  
