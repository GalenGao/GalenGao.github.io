---
layout: post
title:  "oracle12c各个版本对其需要的依赖包及系统参数的修改"
date:   2016-05-26 10:32:04 +0700
categories: [linux, oracle]
---

以下是我在oracle官网上对oracle12c 各个版本的依赖包需求整理

#### 1、Packages for Oracle Linux 7 and Red Hat Enterprise Linux 7  

The following packages (or later versions) must be installed:  

```
binutils-2.23.52.0.1-12.el7.x86_64 
compat-libcap1-1.10-3.el7.x86_64 
gcc-4.8.2-3.el7.x86_64 
gcc-c++-4.8.2-3.el7.x86_64 
glibc-2.17-36.el7.i686 
glibc-2.17-36.el7.x86_64 
glibc-devel-2.17-36.el7.i686 
glibc-devel-2.17-36.el7.x86_64 
ksh
libaio-0.3.109-9.el7.i686 
libaio-0.3.109-9.el7.x86_64 
libaio-devel-0.3.109-9.el7.i686 
libaio-devel-0.3.109-9.el7.x86_64 
libgcc-4.8.2-3.el7.i686 
libgcc-4.8.2-3.el7.x86_64 
libstdc++-4.8.2-3.el7.i686 
libstdc++-4.8.2-3.el7.x86_64 
libstdc++-devel-4.8.2-3.el7.i686 
libstdc++-devel-4.8.2-3.el7.x86_64 
libXi-1.7.2-1.el7.i686 
libXi-1.7.2-1.el7.x86_64 
libXtst-1.2.2-1.el7.i686 
libXtst-1.2.2-1.el7.x86_64 
make-3.82-19.el7.x86_64 
sysstat-10.1.5-1.el7.x86_64 
```

#### 2、Packages for Oracle Linux 6 and Red Hat Enterprise Linux 6  

The following packages (or later versions) must be installed:  

```
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
```

#### 3、Package requirements for Oracle Linux 5 and Red Hat Enterprise Linux 5  

The following packages (or later versions) must be installed:  

```
binutils-2.17.50.0.6
compat-libstdc++-33-3.2.3
compat-libstdc++-33-3.2.3 (32 bit)
coreutils-5.97-23.el5_4.1
gcc-4.1.2
gcc-c++-4.1.2
glibc-2.5-58
glibc-2.5-58 (32 bit)
glibc-devel-2.5-58
glibc-devel-2.5-58 (32 bit)
ksh
libaio-0.3.106
libaio-0.3.106 (32 bit)
libaio-devel-0.3.106
libaio-devel-0.3.106 (32 bit)
libgcc-4.1.2
libgcc-4.1.2 (32 bit)
libstdc++-4.1.2
libstdc++-4.1.2 (32 bit)
libstdc++-devel 4.1.2
libXext-1.0.1
libXext-1.0.1 (32 bit)
libXtst-1.0.1
libXtst-1.0.1 (32 bit)
libX11-1.0.3
libX11-1.0.3 (32 bit)
libXau-1.0.1
libXau-1.0.1 (32 bit)
libXi-1.0.1
libXi-1.0.1 (32 bit) 
make-3.81
sysstat-7.0.2
```

==============================================  

> Oracle Preinstallation RPM for your Oracle Linux 6 kernel (oracle-rdbms-server-12cR1-preinstall).  
> Oracle Validated RPM (oracle-validated) for your Oracle Linux 5 kernel.  

#### before install oracle software needs update OS parameters

`a、vi /etc/security/limits.conf`

For each installation software owner, check the resource limits for installation, using the following recommended ranges:  

**Table 5-1 Installation Owner Resource Limit Recommended Ranges**  

| Resource Shell Limit	                         | Resource | Soft Limit        | Hard Limit                      |
|================================================|==========|===================|=================================|
| Open file descriptors                          | nofile   | at least 1024     | at least 65536                  |
| Number of processes available to a single user | nproc    | at least 2047     | at least 16384                  |
| Size of the stack segment of the process       | stack    | at least 10240 KB | at least 10240 KB, and at most 32768 KB                                                                                                          |
| Maximum Locked Memory Limit | memlock          | at least 90 percent of the current RAM when HugePages memory is enabled and at least 3145728 KB (3 GB) when HugePages memory is disabled | at least 90 percent of the current RAM when HugePages memory is enabled and at least 3145728 KB (3 GB) when HugePages memory is disabled |

**To check resource limits:**  
Log in as an installation owner.  
Check the soft and hard limits for the file descriptor setting. Ensure that the result is in the recommended range, for example:  

```
$ ulimit -Sn
1024
$ ulimit -Hn
65536
```
Check the soft and hard limits for the number of processes available to a user. Ensure that the result is in the recommended range, for example:  

```
$ ulimit -Su
2047
$ ulimit -Hu
16384
```
Check the soft limit for the stack setting. Ensure that the result is in the recommended range, for example:  

```
$ ulimit -Ss
10240
$ ulimit -Hs
32768
```
Repeat this procedure for each Oracle software installation owner.  

> If necessary, update the resource limits in the /etc/security/limits.conf configuration file for the installation owner. However, note that the configuration file is distribution specific. Contact your system administrator for distribution specific configuration file information.  

`vi /etc/security/limits.conf`
{% highlight ruby %}
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 32768

grid soft nproc 2047
grid hard nproc 16384
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 32768
{% endhighlight %}

`b、vi /etc/sysctl.conf`
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
sysctl -p
{% endhighlight %}

`c、vi /etc/profile`
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

`d、vi /etc/pam.d/login`
{% highlight ruby %}
session required /lib/security/pam_limits.so 
session required pam_limits.so
{% endhighlight %}
