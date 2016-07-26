---
layout: post
title:  "CENTOS6.6 下mysql MHA架构搭建"
date:   2016-05-21 14:25:20 +0700
categories: [mysql]
---

**摘要:**  

* **本篇是自己搭建的一篇mysql MHA文章**
* **前面的安装步骤基本不变，后面的比如keepalived的配置文件有几种方法**
* **其实想完成keepalived+lvs+atlas(mycat)+mha+mysql主从复制 这样的架构，只是MYCAT单独文章了**



每个节点都关闭防火墙，SELINUX。

**1、安装epel yum源**    

{% highlight ruby %}
wget http://mirrors.hustunique.com/epel/6/x86_64/epel-release-6-8.noarch.rpm   
wget http://mirrors.hustunique.com/epel/RPM-GPG-KEY-EPEL-6   
rpm --import RPM-GPG-KEY-EPEL-6  
rpm -ivh epel-release-6-8.noarch.rpm 
{% endhighlight %}

**2、所有节点安装MHA node所需的perl模块（DBD:mysql）**  

{% highlight ruby %}
yum -y install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes perl-devel
{% endhighlight %}

**3、建立主从复制**  

* 在node1 master上（node2上也要执行）  
{% highlight ruby %}
grant replication slave  on *.* to 'backup'@'192.168.10.%'identified by'backup';
grant all privileges on *.* to 'root'@'192.168.10.%'identified by'123456';
show master status;
{% endhighlight %}
![mha](/static/img/mha/mha1.png)

* 在node2 node3 slave上  
{% highlight ruby %}
change master  to  master_host='192.168.10.120', master_user='backup', master_password='backup',master_port=3306, master_log_file='mysql-bin.000001',master_log_pos=120,master_connect_retry=1;
start slave;
show slave status \G;
{% endhighlight %}

> 其中Slave_IO_Running 与 Slave_SQL_Running 的值都必须为YES，才表明状态正常。  

**4、ssh-keygen实现三台机器之间相互免密钥登录 在每个节点上都执行下面语句**  

{% highlight ruby %}
ssh-keygen -t rsa
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.120
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.121
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.122
{% endhighlight %}


**5、三节点安装mha4mysql-node-0.56,node3上安装mha4mysql-manager-0.56**  

* 在node1 node2 node3安装mha4mysql-node  
{% highlight ruby %}
wget https://googledrive.com/host/0B1lu97m8-haWeHdGWXp0YVVUSlk/mha4mysql-node-0.56.tar.gz
tar xf mha4mysql-node-0.56.tar.gz
cd mha4mysql-node-0.56
perl Makefile.PL
make && make install
{% endhighlight %}

> 注：若运行 perl Makefile.PL时报Can't locate CPAN.pm in @INC这个错时，表示没有安装CPAN模块：  
> 
> {% highlight ruby %}
> wget http://www.cpan.org/authors/id/A/AN/ANDK/CPAN-2.10.tar.gz
> tar -zxvf CPAN-2.10.tar.gz
> cd CPAN-2.10
> perl Makefile.PL
> make
> make install
> {% endhighlight %}


* 在node3上安装mha4mysql-manager  
{% highlight ruby %}
# 首先安装依赖包 
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Config-IniFiles perl-Time-HiRes
wget https://googledrive.com/host/0B1lu97m8-haWeHdGWXp0YVVUSlk/mha4mysql-manager-0.56.tar.gz
tar xf mha4mysql-manager-0.56.tar.gz
cd mha4mysql-manager-0.56
perl Makefile.PL
make && make install
{% endhighlight %}

* 在node3上管理MHA配置文件
{% highlight ruby %}
mkdir -p /etc/mha/{app1,scripts}
cp -r mha4mysql-manager-0.56/samples/conf/* /etc/mha/
cp -r mha4mysql-manager-0.56/samples/scripts/* /etc/mha/scripts/
mv /etc/mha/app1.cnf /etc/mha/app1/
mv /etc/mha/masterha_default.cnf /etc/masterha_default.cnf
{% endhighlight %}

**6、在manager上设置全局配置：**  

{% highlight ruby %}
vim /etc/masterha_default.cnf

[server default]
user=root
password=123456
ssh_user=root
repl_user=backup
repl_password=backup
ping_interval=1
#shutdown_script=""
secondary_check_script=masterha_secondary_check-s node1 -s node2 -s node3 --user=root --master_host=node1 --master_ip=192.168.10.120 --master_port=3306
#master_ip_failover_script="/etc/mha/scripts/master_ip_failover"
#master_ip_online_change_script="/etc/mha/scripts/master_ip_online_change"
# shutdown_script= /script/masterha/power_manager
#report_script=""
{% endhighlight %}
 

{% highlight ruby %}
# 创建日志目录
mkdir -p /var/log/mha/app1

vim /etc/mha/app1/app1.cnf

# master_binlog_dir 要是mysql的binlog目录，否则验证会报错
[server default]
manager_workdir=/var/log/mha/app1
manager_log=/var/log/mha/app1/manager.log
[server1]
hostname=node1
master_binlog_dir="/usr/local/mysql/logs"
candidate_master=1
[server2]
hostname=node2
master_binlog_dir="/usr/local/mysql/logs"
candidate_master=1
[server3]
hostname=node3
master_binlog_dir="/usr/local/mysql/logs"
no_master=1
{% endhighlight %}


> 注释：  
candidate_master=1 表示该主机优先可被选为new master，当多个[serverX]等设置此参数时，优先级由[serverX]配置的顺序决定;  
secondary_check_script mha强烈建议有两个或多个网络线路检查MySQL主服务器的可用性。默认情况下,只有单一的路线 MHA Manager检查:从Manager to Master,但这是不可取的。MHA实际上可以有两个或两个以上的检查路线通过调用外部脚本定义二次检查脚本参数;  
master_ip_failover_script 在MySQL从服务器提升为新的主服务器时，调用此脚本，因此可以将vip信息写到此配置文件;  
master_ip_online_change_script 使用masterha_master_switch命令手动切换MySQL主服务器时后会调用此脚本，参数和master_ip_failover_script 类似，脚本可以互用   shutdown_script 此脚本(默认samples内的脚本)利用服务器的远程控制IDRAC等，使用ipmitool强制去关机，以避免fence设备重启主服务器，造成脑列现象;  
report_script 当新主服务器切换完成以后通过此脚本发送邮件报告，可参考使用 <http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v1.56.tar.gz>  
以上涉及到的脚本可以从mha4mysql-manager-0.56/samples/scripts/*拷贝进行修改使用  
其他manager详细配置参数<https://code.google.com/p/mysql-master-ha/wiki/Parameters>  

**7、masterha_check_ssh验证ssh信任登录是否成功,masterha_check_repl验证mysql复制是否成功**  

验证ssh信任：  
{% highlight ruby %} 
masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
{% endhighlight %}
![mha](/static/img/mha/mha2.png)  

验证主从复制：  
{% highlight ruby %}
masterha_check_repl --conf=/etc/mha/app1/app1.cnf
{% endhighlight %}
![mha](/static/img/mha/mha3.png)

**8、启动MHA manager，并监控日志文件**  

在node1上killall mysqld的同时在node3上启动manager服务  
{% highlight ruby %}
masterha_manager --conf=/etc/mha/app1/app1.cnf
{% endhighlight %}
![mha](/static/img/mha/mha4.png)

之后观察node3上/var/log/mha/app1/manager.log日志会发现node1 dead状态，主自动切换到node2上，而node3上的主从配置指向了node2，
并且发生一次切换后会生成/var/log/mha/app1/app1.failover.complete文件；  
![mha](/static/img/mha/mha5.png)

手动恢复node1操作：  
{% highlight ruby %}
rm -rf /var/log/mha/app1/app1.failover.complete
{% endhighlight %}

在node1上启动mysql:  
{% highlight ruby %}
service mysql start
{% endhighlight %}

登录进去show master status;
![mha](/static/img/mha/mha6.png)  

重新配置node2 node3 主从指向node1（change master to）  
{% highlight ruby %}
stop slave;
change master  to  master_host='192.168.10.120', master_user='backup', master_password='backup',master_port=3306, master_log_file='mysql-bin.000002',master_log_pos=120,master_connect_retry=1;
start slave;
show slave status\G;
{% endhighlight %}

MHA Manager后台执行：  
(1)查看Manager的状态  
{% highlight ruby %}
masterha_check_status --conf=/etc/mha/app1/app1.cnf
{% endhighlight %}

(2)后台启动  
{% highlight ruby %}
nohup masterha_manager --conf=/etc/mha/app1/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/masterha/app1/manager.log 2>&1 &
{% endhighlight %}

> 启动参数介绍：  
    - –remove_dead_master_conf      该参数代表当发生主从切换后，老的主库的ip将会从配置文件中移除。  
    - –manger_log                   日志存放位置  
    - –ignore_last_failover         在缺省情况下，如果MHA检测到连续发生宕机，且两次宕机间隔不足8小时的话，则不会进行  Failover，之所以这样限制是为了避免ping-pong效应。该参数代表忽略上次MHA触发切换产生的文件，默认情况下，MHA发生切换后会在日志目录，也就是上面我设置的/data产生app1.failover.complete文件，下次再次切换的时候如果发现该目录下存在该文件将不允许触发切换，除非在第一次切换后收到删除该文件，为了方便，这里设置为–ignore_last_failover  

关闭MHA Manage监控  
{% highlight ruby %}
masterha_stop --conf=/etc/mha/app1/app1.cnf
{% endhighlight %}

守护进程方式参考：   
<https://code.google.com/p/mysql-master-ha/wiki/Runnning_Background
ftp://ftp.pbone.net/mirror/ftp5.gwdg.de/pub/opensuse/repositories/home:/weberho:/qmailtoaster/openSUSE_Tumbleweed/x86_64/daemontools-0.76-5.3.x86_64.rpm>

**9、配置VIP**  

vip配置可以采用两种方式，一种通过keepalived的方式管理虚拟ip的漂移；另外一种通过脚本方式启动虚拟ip的方式（即不需要keepalived或者heartbeat类似的软件）。这里采用的是keepalived的方式管理vip的漂移。  
keepalived方式管理虚拟ip，keepalived配置方法如下：  

* 安装keepalived   

{% highlight ruby %}
# 下载软件进行并进行安装（两台master，准确的说一台是master，另外一台是备选master，在没有切换以前是slave）：  
# 首先安装内核包  
yum install kernel-headers kernel-devel
# 然后安装依赖包
yum install popt-static kernel-devel make gcc openssl-devel lftp libnl* popt*
# 最后软连接
ln -s /usr/src/kernels/2.6.32-279.el6.i686//usr/src/linux/
# 安装keepalived
wget  http://www.keepalived.org/software/keepalived-1.2.4.tar.gz
tar zxvf keepalived-1.2.4.tar.gz
cd keepalived-1.2.4
./configure --prefix=/usr/local/keepalived
make
make install

# 将keepalived做成启动服务，方便管理
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
mkdir /etc/keepalived/
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/

# 启动keepalived服务  
service keepalived start
{% endhighlight %}

* 配置keepalived配置文件  
在keepalived中2种模式，分别是master->backup模式和backup->backup模式。这两种模式有很大区别。在master->backup模式下，一旦主库宕机，虚拟ip会自动漂移到从库，当主库修复后，keepalived启动后，还会把虚拟ip抢占过来，即使设置了非抢占模式（nopreempt）抢占ip的动作也会发生。在backup->backup模式下，当主库宕机后虚拟ip会自动漂移到从库上，当原主库恢复和keepalived服务启动后，并不会抢占新主的虚拟ip，即使是优先级高于从库的优先级别，也不会发生抢占。为了减少ip漂移次数，通常是把修复好的主库当做新的备库。下面依次是两种配置方式：  
**a、master->backup模式**  
{% highlight ruby %}
# 配置keepalived的配置文件，在master上配置
[root@192.168.3.110~]# cat /etc/keepalived/keepalived.conf
! ConfigurationFile for keepalived
 
global_defs {
notification_email {
343838596@qq.com
}
notification_email_from 343838596@qq.com
smtp_server 127.0.0.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}
#这里加一个检查mysql服务是否停掉的脚本，如果停掉就关闭keepalived服务器，让其自动跳到另外一台虚拟ip
varrp_script check_mysql{
     script "/etc/keepalived/check_mysql.sh"
}

vrrp_instance VI_1 {
state BACKUP
interface eth0
virtual_router_id 51
priority 80
advert_int 1
nopreempt
authentication {
auth_type PASS
auth_pass 1111
}
virtual_ipaddress {
192.168.10.219
}
}
{% endhighlight %}
> 其中router_id MySQL HA表示设定keepalived组的名称，将192.168.3.250这个虚拟ip绑定到该主机的eth0网卡上，并且设置了状态为backup模式，将keepalived的模式设置为非抢占模式（nopreempt），priority 80表示设置的优先级为80。下面的配置略有不同，但是都是一个意思。 
{% highlight ruby %} 
# 在候选master上配置keepalived的配置文件
cat /etc/keepalived/keepalived.conf 
 
! ConfigurationFile for keepalived
 global_defs {
    notification_email {
    343838596@qq.com
 }
    notification_email_from 343838596@qq.com
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
  }
#这里加一个检查mysql服务是否停掉的脚本，如果停掉就关闭keepalived服务器，让其自动跳到另外一台虚拟ip 
varrp_script check_mysql{
     script "/etc/keepalived/check_mysql.sh"
}

 vrrp_instance VI_1 {
     state BACKUP
  interface eth0
      virtual_router_id 51
       priority 60
       advert_int 1
     nopreempt
      authentication {
         auth_type PASS
         auth_pass 1111
 }
    virtual_ipaddress {
 192.168.10.219
 }
 }
{% endhighlight %} 
**b、backup->backup模式**  
{% highlight ruby %} 
# 配置keepalived的配置文件，在master上配置（node1）

vim /etc/keepalived/keepalived.conf
! ConfigurationFile for keepalived
global_defs{
router_id MHA
notification_email{
root@localhost   #接收邮件，可以有多个，一行一个
}
#当主、备份设备发生改变时，通过邮件通知
notification_email_from  m@localhost
#发送邮箱服务器
smtp_server 127.0.0.1
#发送邮箱超时时间
smtp_connect_timeout 30
}
 
varrp_script check_mysql{
     script "/etc/keepalived/check_mysql.sh"
}
vrrp_sync_group VG1{
    group{
          VI_1
    }
notify_master "/etc/keepalived/master.sh"
}
 
vrrp_instance VI_1{
     state master     
     interface eth0   
     virtual_router_id 110
     priority 100            
     advert_int 1
     nopreempt #不抢占资源，意思就是它活了之后也不会再把主抢回来
 
     authentication{
     # 认证方式，可以是PASS或AH两种认证方式
     auth_type PASS
     # 认证密码
     auth_pass 111111
     }
track_script{
     check_mysql
}
virtual_ipaddress{
     192.168.10.219
     }
 
}
{% endhighlight %} 
{% highlight ruby %} 
# 在slave上配置（node2）

vim /etc/keepalived/keepalived.conf
! ConfigurationFile for keepalived
global_defs{
router_id MHA
notification_email{
root@localhost   #接收邮件，可以有多个，一行一个
}
#当主、备份设备发生改变时，通过邮件通知
notification_email_from  m@localhost
#发送邮箱服务器
smtp_server 127.0.0.1
#发送邮箱超时时间
smtp_connect_timeout 30
}
 
varrp_script check_mysql{
     script "/etc/keepalived/check_mysql.sh"
}
vrrp_sync_group VG1{
    group{
          VI_1
    }
notify_master "/etc/keepalived/master.sh"
}
vrrp_instance VI_1{
     state backup    
     interface eth0    
     virtual_router_id 110
     priority 99            
     advert_int 1
 
     authentication{
     # 认证方式，可以是PASS或AH两种认证方式
     auth_type PASS
     # 认证密码
     auth_pass 111111
     }
track_script{
     check_mysql
}
virtual_ipaddress{
     192.168.10.219
     }
 
}
{% endhighlight %} 
**check_mysql.sh 脚本**  
{% highlight ruby %}
vi /etc/keepalived/check_mysql.sh
#!/bin/bash
MYSQL=/usr/local/mysql/bin/mysql
MYSQL_HOST=127.0.0.1
MYSQL_USER=root
MYSQL_PASSWORD=123456
CHECK_TIME=3
#mysql  is working MYSQL_OK is 1 , mysql down MYSQL_OK is 0
MYSQL_OK=1
function check_mysql_helth(){
$MYSQL -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASSWORD -e "show status;" >/dev/null 2>&1
if[ $? = 0 ];then
     MYSQL_OK=1
else
     MYSQL_OK=0
fi
     return $MYSQL_OK
}
while [ $CHECK_TIME -ne 0 ]
do
     let "CHECK_TIME -= 1"
     check_mysql_helth
if [ $MYSQL_OK = 1 ];then
     CHECK_TIME=0
     exit0
fi
if[ $MYSQL_OK -eq 0 ] &&  [ $CHECK_TIME -eq 0 ]
then
     pkill keepalived
exit 1
fi
sleep 1
done
{% endhighlight %}
**master.sh 脚本**  
{% highlight ruby %}
vi /etc/keepalived/master.sh
#!/bin/bash
VIP=192.168.10.219
GATEWAY=1.1
/sbin/arping -I eth0 -c 5 -s $VIP $GATEWAY &amp;&gt;/dev/null
 
chmod +x  /etc/keepalived/check_mysql.sh
chmod +x  /etc/keepalived/master.sh
{% endhighlight %}

* 启动keepalived服务，在master上启动并查看日志  
{% highlight ruby %}
[root@]# /etc/init.d/keepalived start
Starting keepalived:[  OK  ]
[root@]# tail -f /var/log/messages
{% endhighlight %}

* 查看绑定情况  
{% highlight ruby %}
[root@]# ip addr | grep eth0
2: eth0:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 192.168.3.110/24 brd 192.168.3.255 scope global eth0
    inet 192.168.3.250/32 scope global eth0
[root@]# 
{% endhighlight %}
 
* MHA引入keepalived  
要想把keepalived服务引入MHA，我们只需要修改切换是触发的脚本文件master_ip_failover即可，在该脚本中添加在master发生宕机时对keepalived的处理。其实可以不用master_ip_failover，因为上面配置文件里嵌套了check_mysql脚本，若没有，就要用该脚本。  
编辑脚本/usr/local/bin/master_ip_failover，修改后如下，这里完整贴出该脚本（192.168.3.123）。  
在MHA Manager修改脚本修改后的内容如下：  
{% highlight ruby %}
#!/usr/bin/env perl
 
use strict;
use warnings FATAL =>'all';
 
useGetopt::Long;
 
my(
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
 
my $vip ='192.168.3.219';
my $ssh_start_vip ="/etc/init.d/keepalived start";
my $ssh_stop_vip ="/etc/init.d/keepalived stop";
 
GetOptions(
'command=s'=> \$command,
'ssh_user=s'=> \$ssh_user,
'orig_master_host=s'=> \$orig_master_host,
'orig_master_ip=s'=> \$orig_master_ip,
'orig_master_port=i'=> \$orig_master_port,
'new_master_host=s'=> \$new_master_host,
'new_master_ip=s'=> \$new_master_ip,
'new_master_port=i'=> \$new_master_port,
);
 
exit &main();
 
sub main {
 
print"\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
 
if( $command eq "stop"|| $command eq "stopssh"){
 
my $exit_code =1;
eval{
print"Disabling the VIP on old master: $orig_master_host \n";
&stop_vip();
            $exit_code =0;
};
if($@){
            warn "Got Error: $@\n";
exit $exit_code;
}
exit $exit_code;
}
elsif( $command eq "start"){
 
my $exit_code =10;
eval{
print"Enabling the VIP - $vip on the new master - $new_master_host \n";
&start_vip();
            $exit_code =0;
};
if($@){
            warn $@;
exit $exit_code;
}
exit $exit_code;
}
elsif( $command eq "status"){
print"Checking the Status of the script.. OK \n";
#`ssh $ssh_user\@cluster1 \" $ssh_start_vip \"`;
exit0;
}
else{
&usage();
exit1;
}
}
 
# A simple system call that enable the VIP on the new mastersub start_vip() {
`ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
# A simple system call that disable the VIP on the old_master
sub stop_vip(){
return0unless($ssh_user);
`ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
 
sub usage {
print
"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
{% endhighlight %}
现在已经修改这个脚本了，我们现在打开在上面提到过的参数，再检查集群状态，看是否会报错。  
{% highlight ruby %}
# node3上把注释取消
[root@192.168.3.123~]# grep 'master_ip_failover_script' /etc/masterha/app1.cnf
master_ip_failover_script=/usr/local/bin/master_ip_failover
[root@192.168.3.123~]# 
#在node3上验证主从复制
 masterha_check_repl --conf=/etc/masterha/app1.cnf 
{% endhighlight %}

**10、MHA常用命令**  

{% highlight ruby %}
# 查看manager状态
masterha_check_status --conf=/etc/mha/app1/app1.cnf
# 查看免密钥是否正常
masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
# 查看主从复制是否正常
masterha_check_repl --conf=/etc/mha/app1/app1.cnf
# 添加新节点server4到配置文件
masterha_conf_host --command=add —conf=/etc/mha/app1/app1.cnf —hostname=geekwolf —block=server4 —params=“no_master=1;ignore_fail=1” 
# 删除server4节点
masterha_conf_host --command=delete —conf=/etc/mha/app1/app1.cnf —block=server4
# 注：block:为节点区名，默认值 为[server_$hostname],如果设置成block=100，则为[server100] params:参数，分号隔开(参考https://code.google.com/p/mysql-master-ha/wiki/Parameters)
# 关闭manager服务
masterha_stop —conf=/etc/mha/app1/app1.cnf
# 主手动切换(前提不要启动masterha_manager服务)，在主node1存活情况下进行切换
# 交互模式：
masterha_master_switch —master_state=alive —conf=/etc/mha/app1/app1.cnf —new_master_host=node2
# 非交互模式：
masterha_master_switch —master_state=alive —conf=/etc/mha/app1/app1.cnf —new_master_host=node2 —interactive=0
# 在主node1宕掉情况下进行切换
masterha_master_switch —master_state=dead —conf=/etc/mha/app1/app1.cnf —dead_master_host=node1 —dead_master_ip=192.168.10.216 —dead_master_port=3306 —new_master_host=192.168.10.217 
{% endhighlight %}

> 详细请参考:<https://code.google.com/p/mysql-master-ha/wiki/TableOfContents?tm=6>

**11、注意事项**  

A. 以上两种vip切换方式，建议采用第一种方法；  
B. 发生主备切换后，manager服务会自动停掉，且在/var/log/mha/app1下面生成
app1.failover.complete，若再次发生切换需要删除app1.failover.complete文件；  
C. 测试过程发现一主两从的架构(两从都设置可以担任主角色candidate_master=1)，当旧主故障迁移到备主后，删除app1.failover.complete，再次启动manager，停掉新主后，发现无法正常切换(解决方式：删除/etc/mha/app1/app1.cnf里面的旧主node1的信息后，重新切换正常)；  
D. arp缓存导致切换VIP后，无法使用问题；  
E. 使用Semi-Sync能够最大程度保证数据安全；  
F. Purge_relay_logs脚本删除中继日志不会阻塞SQL线程，在每台从节点上设置计划任务定期清除中继日志  
{% highlight ruby %}
0 5 * * * root /usr/bin/purge_relay_logs —user=root —password=geekwolf —disable_relay_log_purge >> /var/log/mha/purge_relay_logs.log 2>&1
{% endhighlight %}

**12、部署过程遇到的问题**  

问题1：  
{% highlight ruby %}
[root@node1 mha4mysql-node-0.56]# perl Makefile.PL
Can’t locate ExtUtils/MakeMaker.pm in @INC (@INC contains: inc /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/Makefile.pm line 4.
BEGIN failed—compilation aborted at inc/Module/Install/Makefile.pm line 4. Compilation failed in require at inc/Module/Install.pm line 283.
Can’t locate ExtUtils/MakeMaker.pm in @INC (@INC contains: inc /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/
vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/Can.pm line 6.
BEGIN failed—compilation aborted at inc/Module/Install/Can.pm line 6.
Compilation failed in require at inc/Module/Install.pm line 283.
Can’t locate ExtUtils/MM_Unix.pm in @INC (@INC contains: inc /usr/local/lib64/
perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/
Metadata.pm line 349.
{% endhighlight %}

解决办法：  
{% highlight ruby %}
yum -y install perl-CPAN perl-devel perl-DBD-MySQL
{% endhighlight %}

问题2：  
{% highlight ruby %}
Can’t locate Time/HiRes.pm in @INC (@INC contains: /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at /usr/local/share/perl5/MHA/SSHCheck.pm line 28.
BEGIN failed—compilation aborted at /usr/local/share/perl5/MHA/SSHCheck.pm line 28.
Compilation failed in require at /usr/local/bin/masterha_check_ssh line 25. BEGIN failed—compilation aborted at /usr/local/bin/masterha_check_ssh line 25.
{% endhighlight %}

解决办法：  
{% highlight ruby %}
yum -y install perl-Time-HiRes
{% endhighlight %}

问题3：  
![mha](/static/img/mha/mha7.jepg)

解决办法:  
每个节点都做好mysql命令的软链  
{% highlight ruby %}
ln -s /usr/local/mysql/bin/* /usr/local/bin/
{% endhighlight %}






