---
layout: post
title:  "elasticsearch2.3.3安装"
date:   2016-05-30 15:22:04
categories: [elasticsearch]
---
 
**摘要:**  

* **作者原来搭建的环境是0.95版本**
* **现在升级到2.3.3版本，变了很多，也重新安装了一遍**

### maven安装
* **因为后面安装ik插件需要打包，所以先安装maven**

{% highlight ruby %}
# 下载maven软件并解压
wget http://mirrors.cnnic.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
tar -zxvf apache-maven-3.2.1-bin.tar.gz -C /usr/local
mv apache-maven-3.2.1 maven
# 配置环境变量
vi .bash_profile
export M2_HOME=/usr/local/maven
export PATH=$PATH:$JAVA_HOME/bin:$M2_HOME/bin
# 使环境变量生效
source .bash_profile
{% endhighlight %}

### 安装es    
* **下载elasticsearch-2.3.3版本并解压**  

{% highlight ruby %} 
wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.3/elasticsearch-2.3.3.tar.gz
tar -zxvf elasticsearch-2.3.3.tar.gz -C /usr/local
# 重命名
mv elasticsearch-2.3.3 es
{% endhighlight %}  

* **修改配置文件** 

{% highlight ruby %}
# 进入配置文件目录
cd es/config
# 修改
vi elasticsearch.yml

# 起个集群名
 cluster.name: galen
# 起个节点名
 node.name: node-1
# 指定服务器IP
network.host: 192.168.10.145
# 指定服务器端口
 http.port: 9200
# 数据目录和日志目录可以用默认的，不修改
path.data: /path/to/data
path.logs: /path/to/logs
# 搭建集群的时候在使用
discovery.zen.ping.unicast.hosts: ["192.168.10.145", "192.168.10.168"]
{% endhighlight %}

### 安装head插件  
* **head，一款H5的数据查看客户端：**  

{% highlight ruby %}
cd /usr/local/es/bin/
./plugin install mobz/elasticsearch-head
{% endhighlight %}

* **执行完后重启，访问路径**

{% highlight ruby %}
http://192.168.10.145:9200/_plugin/head/
{% endhighlight %}

### 安装ik分词库

从这直接下载ik包，注意对应es版本
https://github.com/medcl/elasticsearch-analysis-ik
此处下载elasticsearch-analysis-ik-1.9.3.zip
* **上传elasticsearch-analysis-ik-1.9.3.zip到linux解压**

{% highlight ruby %}
unzip elasticsearch-analysis-ik-1.9.3.zip
mv elasticsearch-analysis-ik-1.2.6 esik
{% endhighlight %}

* **把ik包下面的ik文件拷入es的config下面**

```
cp -rf ik es/config/
```

* **maven ik的源码包**

{% highlight ruby %}
cd esik
mvn clean package
{% endhighlight %}

* **把打包后的文件解压拷入es的plugins下面

{% highlight ruby %}
# 打包后esik目录下面有个target/release目录
cd esik/target/release
unzip elasticsearch-analysis-ik-1.9.3.zip
# 在es的plugins目录下新建一个ik文件夹，注意plugins文件是当你执行./plugin安装插件时会自动生成
mkdir -p es/plugins/ik
cp -rf * es/plugins/ik
# 可以把那个解压的源文件删除
rm -rf elasticsearch-analysis-ik-1.9.3.zip
# 最后在elasticsearch.yml后面增加一句
index.analysis.analyzer.ik.type : "ik"
# 重启es后生效
./elasticsearch -d
{% endhighlight %}

### 自定义分词
如果分词不够或者达不到需求，可以自定义分词
进入/es/config/ik/custom/下面，建一个.dic结尾的文件，输入你的分词
然后后退一步修改配置文件,重启后生效

{% highlight html %}
vi IKAnalyzer.cfg.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">  
<properties>  
	<comment>IK Analyzer 扩展配置</comment>
tom/mydict.dic;custom/sin
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">cu
sgle_word_low_freq.dic;custom/你建的文件.dic</entry> 	
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords">custom/ext_stopword.dic</entry> 	
</properties>
{% endhighlight %}

### 不安装ik前后的分词地址
* **不安装ik时，访问默认分词，地址为：**  
* 
```
curl -XPOST  "http://localhost:9200/userinfo/_analyze?analyzer=standard&pretty=true&text=我是中国人"
```

* **安装ik后，访问ik分词，地址为:**  

```
curl -XPOST  "http://localhost:9200/userinfo/_analyze?analyzer=ik&pretty=true&text=我是中国人"
```



