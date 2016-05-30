---
title: "CENTOS6.6下redis3.2集群搭建"
layout: post
category: nosql
tags: [redis]
excerpt: "Redis 集群是一个提供在多个Redis间节点间共享数据的程序集.redis3.0以前，只支持主从同步的，如果主的挂了，写入就成问题了。3.0出来后就可以很好帮我们解决这个问题。"
---




[参考：]<http://blog.csdn.net/zhu_tianwei/article/details/44928779>  
Redis3.0 最大的特点就是有了cluster的能力，使用redis-trib.rb工具可以轻松构建Redis Cluster。Redis Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。节点之间使用gossip协议传播信息以及发现新节点，这种结构和Cassandra很相似，Cassandra节点可以转发请求。Redis集群中节点不作为client请求的代理，client根据node返回的错误信息重定向请求。  
# 集群特性  
1.数据可以在cluster的多个node之间进行共享；  
2.一次请求处理多批key的命令将不再被支持，因为这些命令处理的key可能在不同的node之间，使用了它们反而会降低cluster的性能；  
3.提供高HA，即某个node failed后cluster依旧提供高可用性。  
cluster提供如下能力保证：  
1.在cluster内自动把数据划分到不同的set上；  
2.当集群中一小群机器出现网络故障时或者其他种类的failure时，cluster要保证系统继续可用；  
redis的每个node启动后占用两个port 6379 & 16379。redis通过port 6379继续对client提供服务，client通过redis独有的文本协议与node进行通信，所以这个port被成为client port or command port。redis node通过port 16379与cluster内部的其他node进行二进制形式的通信，所以被称为data port or bus port。通过port 16379，node之间进行 failure detection(探活)、configure update(配置更新)、failure authorization(失败确认)。如果node使用别的端口作为command port，那么data port 一定是command port + 10000。
两个不同的cluster之间也可以通过data port进行data migration。  
# 分片
redis cluster内部没用提供一致性hash算法来保证集群的可伸缩能力，而是通过简单的crc16 hash算法来进行sharding，所以它最多提供16384个slot。如果cluster有三个node，分别为 A and B and C，则A负责0 - 5500 slots，B负责5501 - 11000 slots，C负责11001 - 16383 slots。进行扩容的时候，就得在不同的node之间进行slots的迁移，不需要关机，也不会出现服务不可用现象。  
cluster内部每个node（也成为一个instance）由一个master和多个slave构成，当master fail的时候，可以通过选举机制选出一个slave代替master。  
Redis Cluster不提供强一致性。例如cluster接受了一个写请求，给client返回ok，这个写请求的内容也可能丢失。因为其写流程如下：  
1 master B接受了一个写请求；  
2 B写成功，返回ok给client；  
3 B把数据广播给slaves（B1、B2、B3）  
如果第二步执行完毕后，B crash了，则会发生数据不一致现象。这与传统的DBMS类似，它们接收了写请求后，每隔1S才会把数据写入disk，这么做也是在性能和一致性之间做一个平衡。  
如果用户对数据的一致性要求比较高，Redis可能也会兼顾这种需求，将来会提供相应的选项，让redis中的slave没用成功的接受数据之前不会给client返回ok给client。即先执行step 3，然后再执行step 2。  
一致性还有一种场景。假设有client Z，与cluster内各个node A and B and C，以及各个node的replica A1 and B1 and C1，Z与B之间连接正常，但是B与B1以及cluster内其他nodes连接失败。如果Z发起write request，那么B会给他返回ok，但是B1无法获取到相应的数据，这就要求写的时候也要把node与cluster内其他的成员的探活也要考虑在内。基本要求就是，写时间周期要大于探活时间周期（node timeout）。当node B timeout之后，master B会自动进入failing状态，拒绝外部client的连接请求，而cluster则会选出slave B1来代替B。  
# 安装配置redis3.2
## 安装ruby环境
由于通过redis-trib.rb工具构建Redis Cluster，需要rudy环境，执行如下命令安装：  

	yum -y  install zlib ruby rubygems  

安装ruby 的redis库：

	gem install redis

## 安装配置redis3.2
这里我用两台服务器，6个节点，互为主从，即3个主节点3个从节点192.168.10.120和192.168.10.121  
分别在两台上安装redis  
{% highlight shell %}
    wget http://download.redis.io/releases/redis-3.2.0.tar.gz
    tar -zxvf redis-3.2.0.tar.gz
    mkdir redis
    cd redis-3.2.0
    make PREFIX=/usr/local/redis
    make PREFIX=/usr/local/redis install
{% endhighlight %}  
将集群工具复制到/usr/local/redis/bin下  

	cp /usr/local/redis-3.2.0/src/redis-trib.rb /usr/local/redis/bin/

创建数据配置目录  

	mkdir -p /usr/local/redis/{conf,data,logs}

两台都配置  

    cd /usr/local/redis
    cp /usr/local/redis-3.2.0/redis.conf ./conf/redis-6380.conf
    cp /usr/local/redis-3.2.0/redis.conf ./conf/redis-6381.conf
    cp /usr/local/redis-3.2.0/redis.conf ./conf/redis-6382.conf

分别在两台服务器上修改这三个文件，如下，仅仅改对应的端口数字即可：  

    # 基本配置
    daemonize  yes
    pidfile /usr/local/redis/data/redis-6380.pid
    port 6380
    bind 192.168.10.120
    unixsocket /usr/local/redis/data/redis-6380.sock
    unixsocketperm 700
    timeout 300
    loglevel verbose
    logfile /usr/local/redis/logs/redis-6380.log
    databases 16
    dbfilename dump-6380.rdb
    dir /usr/local/redis/data/ 
    
    # aof持久化
    appendonly yes
    appendfilename appendonly-6380.aof
    appendfsync everysec
    no-appendfsync-on-rewrite yes
    auto-aof-rewrite-percentage 80-100
    auto-aof-rewrite-min-size 64mb
    lua-time-limit 5000
    
    # 集群配置
    cluster-enabled yes
    cluster-config-file /usr/local/redis/data/nodes-6380.conf 
    cluster-node-timeout 5000 

# 启动
分别把每个节点都启动起来  

    ./bin/redis-server ./conf/redis-6380.conf ;tail -f logs/redis-6380.log
    ./bin/redis-server ./conf/redis-6381.conf ;tail -f logs/redis-6381.log
    ./bin/redis-server ./conf/redis-6382.conf ;tail -f logs/redis-6382.log
# 查看是否都启动了

	ps -ef |grep redis

# 创建集群

     ./bin/redis-trib.rb create --replicas 1 192.168.10.120:6380 192.168.10.120:6381 192.168.10.120:6382 192.168.10.121:6380 192.168.10.121:6381 192.168.10.121:6382


