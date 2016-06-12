---
layout: post
title:  "centos ELK安装"
date:   2016-04-23 16:25:20 +0700
categories: [linux, elk]
---

ELK是进行日志收集分析用的，具体工作、原理、作用自行google。该文只是我简单的一个搭建笔记。  
[参照]：<http://my.oschina.net/itblog/blog/547250>  

# JDK安装 
 
    $ tar zxvf jdk-7u76-linux-x64.tar.gz -C  /usr/local
    $ cd /usr/local
    $ mv jdk1.7.0_21 jdk1.7
    vi ~/.bash_profile
    JAVA_HOME=/usr/local/jdk1.7
    PATH=$PATH:$HOME/bin:/usr/local/mysql/bin:$JAVA_HOME/bin

# 安装ElasticSearch   

    wget https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.2.0/elasticsearch-2.2.0.tar.gz
    tar -zxvf elasticsearch-2.2.0.tar.gz
    cd elasticsearch-2.2.0

安装Head插件（Optional）：  

	./bin/plugin install mobz/elasticsearch-head

然后编辑ES的配置文件：  

    vi config/elasticsearch.yml
    
    cluster.name=es_cluster
    node.name=node0
    path.data=/tmp/elasticsearch/data
    path.logs=/tmp/elasticsearch/logs
    #当前hostname或IP，我这里是centos2
    network.host=centos2
    network.port=9200

其他的选项保持默认，然后启动ES：  

	./bin/elasticsearch

访问：  

    http://192.168.10.141:9200/
    http://192.168.10.141:9200/_plugin/head/

# 安装Logstash

	wget https://download.elastic.co/logstash/logstash/logstash-2.2.2.tar.gz

配置Logstash：  

    tar -zxvf logstash-2.2.2.tar.gz
    cd logstash-2.2.2
    
    mkdir config
    vi config/log4j_to_es.conf

输入以下内容：  

    # For detail structure of this file
    # Set: https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html
    input {
      # For detail config for log4j as input, 
      # See: https://www.elastic.co/guide/en/logstash/current/plugins-inputs-log4j.html
      log4j {
    mode => "server"
    host => "192.168.10.141"
    port => 4567
      }
    }
    filter {
      #Only matched data are send to output.
    }
    output {
      # For detail config for elasticsearch as output, 
      # See: https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html
      elasticsearch {
    action => "index"#The operation on ES
    hosts  => "192.168.10.141:9200" #ElasticSearch host, can be array.
    index  => "ec"   #The index to write data to, can be any string.
      }
    }

因此使用agent来启动它（使用-f指定配置文件）： 

	./bin/logstash agent -f config/log4j_to_es.conf

# 安装Kibana

	wget https://download.elastic.co/kibana/kibana/kibana-4.4.1-linux-x64.tar.gz

配置Kibana:  

	tar -zxvf kibana-4.4.1-linux-x64.tar.gz
	cd kibana-4.4.1-linux-x64

修改以下几项（由于是单机版的，因此host的值也可以使用localhost来代替，这里仅仅作为演示）：  

	vi config/kibana.yml

	server.port: 5601
	server.host: “centos2”
	elasticsearch.url: http://centos2:9200
	kibana.index: “.kibana”

启动kibana：  

	./bin/kibana

*后面用浏览器打开kibana，http://IP:5601 进行一步一步配置*

