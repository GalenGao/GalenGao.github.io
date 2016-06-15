---
layout: post
title:  "Python 脚本实现对 Linux 服务器的监控"
date:   2016-05-27 15:32:04 +0700
categories: [python]
---
 
**摘要:**  

* [原文地址](http://mp.weixin.qq.com/s?__biz=MzA3OTgyMDcwNg==&mid=2650625397&idx=1&sn=2e63b7de09b86dc9c6042c647df44a24&scene=0#wechat_redirect)
* 由于原文来自微信公众号，并且脚本都是图片，所以这里自己亲自把代码整理执行一遍

### 工作原理：基于/proc文件

Linux 系统为管理员提供了非常好的方法，使其可以在系统运行时更改内核，而不需要重新引导内核系统，这是通过/proc 虚拟文件系统实现的。/proc 文件虚拟系统是一种内核和内核模块用来向进程（process）发送信息的机制（所以叫做“/proc”），这个伪文件系统允许与内核内部数据结构交互，获取有关进程的有用信息，在运行中（on the fly）改变设置（通过改变内核参数）。与其他文件系统不同，/proc 存在于内存而不是硬盘中。proc 文件系统提供的信息如下：  

* 进程信息：系统中的任何一个进程，在 proc 的子目录中都有一个同名的进程 ID，可以找到 cmdline、mem、root、stat、statm，以及 status。某些信息只有超级用户可见，例如进程根目录。每一个单独含有现有进程信息的进程有一些可用的专门链接，系统中的任何一个进程都有一个单独的自链接指向进程信息，其用处就是从进程中获取命令行信息。

* 系统信息：如果需要了解整个系统信息中也可以从/proc/stat 中获得，其中包括 CPU 占用情况、磁盘空间、内存对换、中断等。

* CPU 信息：利用/proc/CPUinfo 文件可以获得中央处理器的当前准确信息。

* 负载信息：/proc/loadavg 文件包含系统负载信息。  

* 系统内存信息：/proc/meminfo 文件包含系统内存的详细信息，其中显示物理内存的数量、可用交换空间的数量，以及空闲内存的数量等。

**/proc 目录中的主要文件的说明：**  

|文件或目录名称|描述                                                 |  
|------------|-----------------------------------------------------|  
|apm         |高级电源管理系统                                       |  
|cmdline     |内核文件启动的命令行                                    |  
|CPUinfo     |中央处理器信息                                         |  
|devices     |可以用到的设备（块设备/字符设备）                         |  
|dma         |显示当前使用的DMA通道                                   |  
|filesystems |核心配置的文件系统                                      |  
|ioports     |当前使用的I/O端口                                      |  
|interrupts  |这个文件的每一行都有一个保留终端                          |  
|kcore       |系统物理内存映像                                        |  
|kmsg        |核心输出的消息，被送到日志文件                            |  
|mdstat      |这个文件包括由MD设备驱动程序控制的RAID设备信息             |  
|loadavg     |系统平均负载均衡                                        |  
|meminfo     |存储器使用信息，包括物理内存和交换内存                     |  
|modules     |这个文件给出可加载的内核模块，lsmod程序显示有关模块名称     |  
|net         |网络协议状态信息                                        |  
|partitions  |系统识别的分区表                                        |  
|pci         |pci设备信息                                            |
|scsi        |scsi设备信息                                           |
|self        |到查看/proc程序进程目录的符号链接                         |  
|stat        |包含CPU利用率，内存页、内存兑换、磁盘，全部中断，接触开关自举时间|  
|swaps       |交换分区的使用情况                                       |  
|uptime      |给出系统自从上次自举以来的秒数，以及有多少秒处于空闲         |  
|version     |这个文件只有一行内容，说明运行内核版本                     |  

### Python脚本对linux服务器的监控  

#### 对于CPU的监控

**获取CPU信息**  

* **脚本**
{% highlight python %}
#!/usr/bin/env python

from __future__ import print_function
from collections import OrderedDict
import pprint

def CPUinfo():
    ''' Return the informaton in /proc/CPUinfo
    as a dictionary in the following format:
    CPU_info['proc0']={...}
    CPU_info['proc1']={...}
    '''
    CPUinfo=OrderedDict()
    procinfo=OrderedDict()

    nprocs = 0
    with open('/proc/cpuinfo') as f:
        for line in f:
            if not line.strip():
                # end of one processor
                CPUinfo['proc%s' % nprocs] = procinfo
                nprocs=nprocs+1
                # reset
                procinfo=OrderedDict()
            else:
                if len(line.split(':')) == 2:
                    procinfo[line.split(':')[0].strip()] = line.split(':')[1].strip()
                else:
                    procinfo[line.split(':')[0].strip()] = ''
    return CPUinfo

if __name__=='__main__':
    CPUinfo = CPUinfo()
    for processor in CPUinfo.keys():
        print(CPUinfo[processor]['model name'])

{% endhighlight %}

*  **结果**
{% highlight python %}
[root@cdtest ~]# python cpu.py   
  
Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
{% endhighlight %}

#### 对于系统负载的监控

**获取负载信息** 

* **脚本**  
{% highlight python %}
#!/usr/bin/env python

import os
def load_stat():
    loadavg = {}
    f = open("/proc/loadavg")
    con = f.read().split()
    f.close()
    loadavg['lavg_1']=con[0]
    loadavg['lavg_5']=con[1]
    loadavg['lavg_15']=con[2]
    loadavg['nr']=con[3]
    loadavg['last_pid']=con[4]
    return loadavg
print ("loadavg",load_stat()['lavg_15'])

{% endhighlight %}

*  **结果**
{% highlight python %}
[root@cdtest ~]# python cpu2.py 

('loadavg', '0.00')
{% endhighlight %}

#### 对于系统内存的监控

**获取内存信息** 

* **脚本**  
{% highlight python %}
#!/usr/bin/env python

from __future__ import print_function
from collections import OrderedDict

def meminfo():
    ''' Return the information in /proc/meminfo
    as a dictionary '''
    meminfo=OrderedDict()

    with open('/proc/meminfo') as f:
        for line in f:
            meminfo[line.split(':')[0]] = line.split(':')[1].strip()
    return meminfo

if __name__=='__main__':
    meminfo = meminfo()
    print('Total memory:{0}'.format(meminfo['MemTotal']))
    print('Free memory:{0}'.format(meminfo['MemFree']))

{% endhighlight %}

*  **结果**
{% highlight python %}
[root@cdtest ~]# python cpu3.py   

Total memory:3925652 kB
Free memory:2999584 kB
{% endhighlight %}

#### 对于网络接口的监控

**获取网络信息** 

* **脚本**  
{% highlight python %}
#!/usr/bin/env python

import time
import sys

if len(sys.argv)>1:
    INTERFACE = sys.argc[1]
else:
    INTERFACE = 'eth0'
STATS = []
print ('Interface:',INTERFACE)

def rx():
    ifstat = open('/proc/net/dev').readlines()
    for interface in ifstat:
        if INTERFACE in interface:
            stat = float(interface.split()[1])
            STATS[0:] = [stat]


def tx():
    ifstat = open('/proc/net/dev').readlines()
    for interface in ifstat:
        if INTERFACE in interface:
            stat = float(interface.split()[9])
            STATS[1:] = [stat]

print ('In        Out')
rx()
tx()

while True:
    time.sleep(1)
    rxstat_o = list(STATS)
    rx()
    tx()
    Rx = float(STATS[0])
    Rx_o = rxstat_o[0]
    Tx = float(STATS[1])
    Tx_o = rxstat_o[1]
    RX_RATE = round((Rx-Rx_o)/1024/1024,3)
    TX_RATE = round((Tx-Tx_o)/1024/1024,3)
    print(RX_RATE,'MB     ',TX_RATE,'MB    ')
{% endhighlight %}

*  **结果**
{% highlight python %}
[root@cdtest ~]# python cpu4.py 

('Interface:', 'eth0')
In        Out
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.002, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
(0.0, 'MB     ', 0.0, 'MB    ')
{% endhighlight %}

#### 对于系统某进程的监控

**获取进程信息** 

* **脚本** 
{% highlight python %}
#!/usr/bin/env python

import os,sys,time

while True:
    time.sleep(4)
    try:
        ret = os.popen('ps -C apache -o pid,cmd').readlines()
        if len(ret) < 2:
            print ("apache exit,4s restart")
            time.sleep(3)
            os.system("service apache restart")
    except:
        print("error")

{% endhighlight %}

*  **结果**
该结果要系统上有相应的进程才行，可以根据自己需要监控什么进程修改脚本