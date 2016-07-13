---
layout: post
title:  "linux下简洁优化部署tomcat应用"
date:   2016-06-04 15:25:20 +0700
categories: [tomcat]
---

**摘要:**  

* **本文是自己根据XX大神的优化部署tomcat方法整理出来的文本**



# 修改系统内核

修改linux的一些系统参数，以优化系统性能

* 修改LIMITS.CONF

{% highlight ruby %}
$ vi /etc/security/limits.conf
# 增加
* soft nofile 65536
* hard nofile 65536
{% endhighlight %}

* 修改SYSCTL.CONF

{% highlight ruby %}
# 备份
$ mv /etc/sysctl.conf /etc/sysctl.conf.bak

$ vi /etc/sysctl.conf
# 插入
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024    65000
{% endhighlight %}

 

#	安装APR

* 下载apr-1.5.2.tar.gz和apr-util-1.5.4.tar.gz  
安装apr，方便tomcat 开启aoi   
[apr下载地址](https://apr.apache.org/download.cgi)

{% highlight ruby %}
$ tar -zxvf apr-1.5.2.tar.gz
$ tar -zxvf apr-util-1.5.4.tar.gz
$ yum install gcc gcc-c++ autoconf libtool

# apr安装
$ cd apr-1.5.2
$ ./configure --prefix=/usr/local/apr  
# --prefix=/usr/local/apr指apr的安装路径，以下要用到
$ make  
$ make install

# apr-util安装
$ cd apr-util-1.5.4
$ ./configure --with-apr=/usr/local/apr
# --with-apr=/usr/local/apr 指定APR安装路径
$ make
$ make install
{% endhighlight %}

# 安装JDK1.7

下载jdk-7u76-linux-x64.tar.gz 拷贝到/usr/local下面

{% highlight ruby %}
cd /usr/local
$ tar -zxvf jdk-7u76-linux-x64.tar.gz

# mv 重命名 为jdk1.7.0_76
mv jdk1.7.0_76 jdk1.7

# 在环境变量/etc/profile里配置JAVA_HOME
JAVA_HOME=/usr/local/jdk1.7
PATH=$PATH:$HOME/bin:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
 
# 环境变量立即生效
$ source .bash_profile
{% endhighlight %}

# 编译安装tomcat

## 创建DATA目录

{% highlight ruby %}
$ mkdir /data
{% endhighlight %}

## 复制TOMCAT到目标目录并解压  

{% highlight ruby %}
$ cp apache-tomcat-7.0.64.tar.gz /data
$ tar -zxvf apache-tomcat-7.0.64.tar.gz
# mv 重命名为你所需的项目名称，如zhnx
$ mv apache-tomcat-7.0.64 zhnx
{% endhighlight %}

##	进入TOMCAT精简目录

**请参照下面步骤一步一步顺序操作**  
{% highlight ruby %}
$ cd zhnx

# 把项目除了bin lib conf目录的其它全部删除
$ rm -rf LICENSE logs NOTICE RELEASE-NOTES RUNNING.txt  temp/webapps/work/

# 在同目录新建三个文件夹domain、server、sbin
$ mkdir domain
$ mkdir server
$ mkdir sbin

# 把bin 和lib目录移动到server下面
$ mv bin/ server/
$ mv lib/ server/

# 把conf目录移动到domain下面
$ mv conf/ domain/

# 安装tomcat native 开启aio
# 进入server/bin目录下安装tomcat native 与apr连起来 开启tomcat aio模式
$ cd server/bin

# 解压 tomcat native
$ tar -zxvf tomcat-native.tar.gz

# 进入安装tomcat  native目录
$ cd tomcat-native-1.1.33-src
$ cd jni
$ cd native

# 执行命令
$ ./configure --with-apr=/usr/local/apr --with-java-home=/usr/local/jdk1.7.0_76
$ make
$ make install

# 创建项目目录(这里项目名称示例CENTER）
$ cd /data/zhnx/domain
$ mkdir center

# 把conf目录移动到项目下
$ mv conf/ center/
{% endhighlight %}

**拷贝并修改脚本启动参数**  

* **到目录/data/zhnx/sbin 下新建三个脚本，这里与center项目为例：startcenter.sh、stopcenter.sh和super.sh**  
{% highlight ruby %}
$ cd /data/zhnx/sbin

# 脚本一
$ vi startcenter.sh 
#!/bin/bash

# 下面需要修改的地方
# 1、HOME=/data/zhnx根据自己建的目录
export HOME=/data/zhnx
export DOMAIN_HOME=${HOME}/domain
export LOG_HOME=${HOME}/logs
export TOMCAT_HOME=${HOME}/server

#Variable settings begin
# 下面需要修改的地方
# 1、export projectName=center  服务名称，即war包名称
# 2、export httpPort=7001  启动接收请求端口端口
# 3、export serverPort=8001  tomcat启动的本地端口
export projectName=center
export httpPort=7001
export serverPort=8001
export minMsMem=3200m
export maxMsMem=3200m
export ssMem=300k
export mnMem=1100m
export survivorRatior=2
export minPermSize=250m
export maxPermSize=300m
export threshold=20
export fraction=60
export pageSize=128m
export warFile=${HOME}/center_webapps # 服务名加webapps war包地址
export logFile=${LOG_HOME}/${projectName}/catalina.$(date +'%Y-%m-%d').out
export pidFile=${LOG_HOME}/${projectName}.pid
export LD_LIBRARY_PATH=/usr/local/apr/lib
export heapDumpPath=${DOMAIN_HOME}/${projectName}/heapDump
#Variable settings end

#JVM args settings begin
# 下面需要修改的地方：
# 1、Dtjtag=center服务名称
CATALINA_OPTS="-server -Dtjtag=center -Dtomcat.server.port=${serverPort} -Dtomcat.http.port=${httpPort} -Dtomcat.deploy.home=${warFile}"
CATALINA_OPTS="${CATALINA_OPTS} -Xms${minMsMem} -Xmx${maxMsMem} -Xss${ssMem} -Xmn${mnMem} -XX:SurvivorRatio=${survivorRatior} -XX:PermSize=${minPermSize} -XX:MaxPermSize=${maxPermSize}"
CATALINA_OPTS="${CATALINA_OPTS} -XX:+UseCompressedOops -XX:+TieredCompilation -XX:+AggressiveOpts -XX:+UseBiasedLocking"
CATALINA_OPTS="${CATALINA_OPTS} -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -Xnoclassgc -XX:MaxTenuringThreshold=${threshold} -XX:CMSInitiatingOccupancyFraction=${fraction} -XX:LargePageSizeInBytes=${pageSize} -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${heapDumpPath}"
#JVM args settings end

export CATALINA_BASE=${DOMAIN_HOME}/${projectName}
export CATALINA_OPTS
export CATALINA_OUT="${logFile}"
export CATALINA_PID="${pidFile}"
export CATALINA_OPTS
${TOMCAT_HOME}/bin/catalina.sh start

exit $?
{% endhighlight %}



{% highlight ruby %}
$ cd /data/zhnx/sbin

# 脚本二
$ vi stopcenter.sh 

#!/bin/bash

# 下面需要修改的地方
# 1、HOME=/data/zhnx根据自己建的目录
# 2、JAVA_HOME=/usr/local/jdk1.7 根据自己JDK的路劲
export HOME=/data/zhnx
export JAVA_HOME=/usr/local/jdk1.7
export DOMAIN_HOME=${HOME}/domain
export LOG_HOME=${HOME}/logs
export TOMCAT_HOME=${HOME}/server
export CLASSPATH=.:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=${JAVA_HOME}/bin:${PATH}

#Variable settings begin
# 下面需要修改的地方
# 1、export projectName=center 项目名称，即war包名称
# 2、export serverPort=8001  tomcat本地启动端口，以上面对应
export projectName=center 
export serverPort=8001
#Variable settings end

export JAVA_OPTS="-Dtomcat.server.port=${serverPort}"
export CATALINA_PID="${LOG_HOME}/${projectName}.pid"
export CATALINA_BASE=${DOMAIN_HOME}/${projectName}
${TOMCAT_HOME}/bin/catalina.sh stop 30 -force
exit $?
{% endhighlight %}

{% highlight ruby %}
$ cd /data/zhnx/sbin

# 脚本三
vi super.sh

checkProcess(){
   pid=`ps -ef|grep ${classname}|grep -v 'grep'|awk '{print $2}'`
   if [ "X${pid}" != "X" ]; then
      return 0
   else
      return 1
   fi
}

printStartStatus(){
   checkProcess
   if [ $? -eq 0 ]; then
      echo "${_moduleName} start sucessful.      [OK]"
      return 0
   else
      echo "${_moduleName} start Failed.      [Failed]"
      return 1
   fi
}

startProcess(){
   echo $1
   checkProcess
   if [ $? -eq 0 ]; then
          echo "${_moduleName} had started."
      return 0
   else
      ${start}
      sleep 3 
          printStartStatus
   fi       
}

stopProcess(){
   echo $1
   checkProcess
   if [ $? -eq 1 ]; then
      echo "${_moduleName} not running.      [FAILED]"
   else
      pid=`ps -ef|grep ${classname}|grep -v 'grep'|awk '{print $2}'`
      if [ "X${pid}" = "X" ]; then
          echo "${_moduleName} had stop.       [OK]"
      else
          kill -9 $pid
          echo "${_moduleName} stop sucessful.       [OK]"
      fi
   fi
}

restartProcess(){
   stopProcess $1
   startProcess $2
}

getProcessStatus(){
   checkProcess
   if [ $? -eq 0 ]; then
      echo "${_moduleName} is running."
      return 0
   else
      echo "${_moduleName} is not running."
      return 1
   fi
}

unstall(){
   checkProcess
   if [ $? -eq 0 ]; then
      pid=`ps -ef|grep ${classname}|grep -v 'grep'|awk '{print $2}'`
      if [ "X${pid}" != "X" ]; then
         kill -9 $pid
      fi
   fi
   ${remove}
}

commandError(){
   echo ""
   echo "ERROR:UNKNOWN COMMAND:\"$_command\" "
   exit 1
}
{% endhighlight %}

* **到目录/etc/rc.d/init.d/下新建一个脚本，这里与center项目为例：center**
{% highlight ruby %}
$ cd /etc/rc.d/init.d/

# 脚本四
$ vi center

#!/bin/sh

# 下面需要修改的地方
# 1、classname="tjtag=center" center服务名称
# 2、_moduleName="center" modul名称
# 3、start="/data/zhnx/sbin/startcenter.sh" 上面新建的脚本
# 4、. /data/zhnx/sbin/super.sh 上面新建的脚本
classname="tjtag=center" 
_command=$1
_moduleName="center"
start="/data/zhnx/sbin/startcenter.sh"
. /data/zhnx/sbin/super.sh
case $_command in
   start)
      startProcess "Starting ${_moduldName}:"
   ;;
   stop)
      stopProcess "Stoping ${_moduldName}:"
   ;;
   restart)
      restartProcess "Stoping ${_moduldName}:" "Starting ${_moduldName}:"
   ;;
   status)
      getProcessStatus
   ;;
 *)
   commandError
 ;;
esac
{% endhighlight %}

**删除原始的并替换修改过的SERVER.XML**
{% highlight html %}
$ cd /data/zhnx/domain/center/conf
$ rm -rf server.xml

vi server.xml

<?xml version='1.0' encoding='utf-8'?>
<Server port="${tomcat.server.port}" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="off" />
  <Listener className="org.apache.catalina.core.JasperListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">

  <Connector URIEncoding="UTF-8" minSpareThreads="15" maxSpareThreads="30" enableLookups="false" disableUploadTimeout="true" acceptCount="500" maxThreads="50" maxProcessors="1000" minProcessors="10" useURIValidationHack="false" compression="on" compressionMinSize="512" compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain" protocol="org.apache.coyote.http11.Http11AprProtocol" port="${tomcat.http.port}" connectionTimeout="20000"/>

    <Engine name="Catalina" defaultHost="localhost">

      <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm" resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="${tomcat.deploy.home}"  unpackWARs="true" autoDeploy="false">
        <Context path="" docBase="sso" sessionCookiePath="/" sessionCookieDomain=".qasite.com"  useHttpOnly="true" crossContext="true" debug="0" reloadable="false">
          <Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
          memcachedNodes="n1:192.168.10.11:11220" sticky="false" lockingMode="auto"
          sessionBackupAsync="false" sessionBackupTimeout="1000"
          transcoderFactoryClass="de.javakaffee.web.msm.JavaSerializationTranscoderFactory" />
        </Context>
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
{% endhighlight %}

**若有自己需要增加的jar就拷进去**  
$ cd /data/zhnx/server/lib
把jar考进去
 
# 项目部署


新建目录webapp名称与startcenter.sh里的webapps名称一样  
日志名称与服务名一样  
$ mkdir -p /data/zhnx/center_webapps  

$ mkdir -p /data/zhnx/logs/center  

center.war 部署至 center_webapps  

部署时需将对应webapps目录下的文件及文件夹清空；  
然后把项目war包拷入对应webapps目录下  
启动服务  
{% highlight ruby %} 
service center start
{% endhighlight %}


