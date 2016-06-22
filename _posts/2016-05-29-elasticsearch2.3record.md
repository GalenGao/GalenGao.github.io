---
layout: post
title:  "elasticsearch2.3.3集群搭建踩到的坑"
date:   2016-05-29 16:32:04 +0700
categories: [elasticsearch]
---
 
**摘要:**  

* **作者原来搭建的环境是0.95版本**
* **现在升级到2.3.3版本，变了很多，也踩了很多坑**


1.  版本升级到2.2后，必须建一个单独的账号用于启动elasticsearch，不可以使用root账号进行启动，否则会报以下错误   
  
{% highlight ruby %} 
Exception in thread "main" java.lang.RuntimeException: don't run elasticsearch as root.
{% endhighlight %} 

> 也可以这样解决：  
在bin目录修改`elasticsearch.in.sh`文件，填加如下配置项：  
>
{% highlight ruby %} 
JAVA_OPTS="$JAVA_OPTS -Des.insecure.allow.root=true"
{% endhighlight %} 
>
这样就可以用root用户启动elasticsearch了，当然还是建议大家创建其他用户使用。

2.  如果需要通过ip进行访问es集群，必须修改elasticsearch.yml中的network.host节点。es 0.9版本的默认配置是 "0.0.0.0"，所以不绑定ip也可访问，但是es 2.3版本如果采用默认配置，只能通过 localhost 和 "127.0.0.1"进行访问。  
network.host节点可以配置多个值，如下：  

{% highlight ruby %} 
network.host: 192.168.10.145
{% endhighlight %}

3.  es0.9 版本的集群的discovery默认采用的是组播（multicast）模式，但是在es2.3版本中已去除该模式，虽然提供了multicast的插件，但是官方说不建议采用multicast的模式，故我们只能采用单播(unicast)模式：  

{% highlight ruby %}
discovery.zen.ping.unicast.hosts: ["192.168.10.145", "192.168.10.168"]  
{% endhighlight %}  

关于节点“discovery.zen.ping.unicast.hosts”的值可以是单值也可以是多值，在不同的服务器之间部署es节点可以不指明ip端口，但是在同一服务器中部署，ip最好是加上检测的端口号，否则可能检测不到要加入的节点，如下配置：  

{% highlight ruby %}
discovery.zen.ping.unicast.hosts: ["192.168.10.145:9300"]
{% endhighlight %}
 
4.  启动，es0.9版本直接`./elasticsearch`就后台启动了，而es2.3必须`./elasticsearch -d`才是后台启动，否则是前台启动：  

5.  插件安装，es0.9插件安装是`./plugin -install xxx`,而es2.3插件安装没有减号`./plugin install xxx`

6.  IK插件安装，es0.9版本直接下载IK，`mvn clean package` 打包后拷贝target目录下的jar到plugins目录下新建的IK文件夹下即可；
  但是es2.3是`mvn clean package` 打包后，到target/release目录下，有一个`zip`文件，直接unzip解压，然后把解压的内容拷贝到plugins目录下的ik文件下下，否则会报如下错误：  

{% highlight ruby %}
Exception in thread "main" java.lang.IllegalStateException: Could not load plugin descriptor for existing plugin [analysis-ik]. Was the plugin built before 2.0?
Likely root cause: java.nio.file.NoSuchFileException: /home/es/es2/plugins/analysis-ik/plugin-descriptor.properties
{% endhighlight %}