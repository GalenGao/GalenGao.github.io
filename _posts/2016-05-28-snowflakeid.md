---
layout: post
title:  "snowflake 分布式唯一ID生成器"
date:   2016-05-28 16:32:04 +0700
categories: [mysql,snowflake]
---
 
**摘要:**  

* 原文参考运维生存和开源中国上的代码整理
* 我的环境是python3.5，pip8.2的

## 一、python版本

**前言**  

由于考虑到以后要动态切分数据，防止将不同表切分数据到同一个表中时出现主键相等的冲突情况，这里我们使用一个全局ID生存器。重要的是他是自增的。  
这边我使用`Snowflake`的python实现版（pysnowflake）。当然你也可以使用java实现版.  
具体详细信息：[参考网址](http://pysnowflake.readthedocs.org/en/latest/)  

### Snowflake的使用

* **安装 requests**
 
{% highlight ruby %}
pip install requests
{% endhighlight %}

* **安装 pysnowflake** 

{% highlight ruby %}
pip install pysnowflake
{% endhighlight %}

* **启动pysnowflake服务** 

{% highlight ruby %}
snowflake_start_server \
  --address=192.168.10.145 \
  --port=30001 \
  --dc=1 \
  --worker=1 \
  --log_file_prefix=/tmp/pysnowflask.log

# --address：本机的IP地址默认localhost这里解释一下参数意思（可以通过--help来获取）：
# --dc：数据中心唯一标识符默认为0
# --worker：工作者唯一标识符默认为0
# --log_file_prefix：日志文件所在位置
{% endhighlight %}
 
* **使用示例（这边引用官网的）**  

{% highlight ruby %}
# 导入pysnowflake客户端
>>> import snowflake.client
 
# 链接服务端并初始化一个pysnowflake客户端
>>> host = '192.168.10.145'
>>> port = 30001
>>> snowflake.client.setup(host, port)
# 生成一个全局唯一的ID（在MySQL中可以用BIGINT UNSIGNED对应）
>>> snowflake.client.get_guid()
3631957913783762945
# 查看当前状态
>>> snowflake.client.get_stats()
{
  'dc': 1,
  'worker': 1,
  'timestamp': 1454126885629, # current timestamp for this worker
  'last_timestamp': 1454126890928, # the last timestamp that generated ID on
  'sequence': 1, # the sequence number for last timestamp
  'sequence_overload': 1, # the number of times that the sequence is overflow
  'errors': 1, # the number of times that clock went backward
}
{% endhighlight %}

* **数据整理重建ID**  

重建ID是一个很庞大的工程，首先要很了解表的结构。不然，如果少更新了某个表的一列都会导致数据的不一致。  
当然，如果你的表中有很强的外键以及设置了级联那更新一个主键会更新其他相关联的外键。这里我还是不建议去依赖外键级联更新来投机取巧毕竟如果有数据库的设计在项目的里程碑中经过了n次变化，也不能肯定设置的外键一定是级联更新的。  
在这边我强烈建议重建ID时候讲MySQL中的检查外键的参数设置为0。  
{% highlight ruby %}
SET FOREIGN_KEY_CHECKS=0;
{% endhighlight %}
> 小提示：其实理论上我们是没有必要重建ID的因为原来的ID已经是唯一的了而且是整型，他兼容BIGINT。但是这里我还是做了重建，主要是因为以后的数据一致。并且如果有些人的ID不是整型的，而是有一定含义的那时候也肯定需要做ID的重建。  
 
* **修改相关表ID的数据类型为BIGINT**

{% highlight SQL %}
-- 修改商品表 goods_id 字段
ALTER TABLE goods_1
  MODIFY COLUMN goods_id BIGINT UNSIGNED NOT NULL 
  COMMENT '商品ID';
 
-- 修改出售订单表 goods_id 字段
ALTER TABLE sell_order_1
  MODIFY COLUMN sell_order_id BIGINT UNSIGNED NOT NULL 
  COMMENT '出售订单ID';
 
-- 修改购买订单表 buy_order_id 字段
ALTER TABLE buy_order_1
  MODIFY COLUMN buy_order_id BIGINT UNSIGNED NOT NULL 
  COMMENT '出售订单ID与出售订单相等';
 
-- 修改订单商品表 order_goods_id、orders_id、goods_id 字段
ALTER TABLE order_goods_1
  MODIFY COLUMN order_goods_id BIGINT UNSIGNED NOT NULL 
  COMMENT '订单商品表ID';
ALTER TABLE order_goods_1
  MODIFY COLUMN sell_order_id BIGINT UNSIGNED NOT NULL 
  COMMENT '订单ID';
ALTER TABLE order_goods_1
  MODIFY COLUMN goods_id BIGINT UNSIGNED NOT NULL 
  COMMENT '商品ID';
{% endhighlight %}

* **使用python重建ID**

使用的python 模块：  
  
|模块名	                |版本	|备注            |
|-----------------------|-------|---------------|
|pysnowflake	        |0.1.3	|全局ID生成器     |
|mysql_connector_python	|2.1.3	|mysql python API|

这边只展示主程序：完整的程序在附件中都有

{% highlight python %}
if __name__=='__main__':
  # 设置默认的数据库链接参数
  db_config = {
    'user'    : 'root',
    'password': 'root',
    'host'    : '127.0.0.1',
    'port'    : 3306,
    'database': 'test'
  }
  # 设置snowflake链接默认参数
  snowflake_config = {
    'host': '192.168.137.11',
    'port': 30001
  }
 
  rebuild = Rebuild()
  # 设置数据库配置
  rebuild.set_db_config(db_config)
  # 设置snowflak配置
  rebuild.set_snowflake_config(snowflake_config)
  # 链接配置snowflak
  rebuild.setup_snowflake()
 
  # 生成数据库链接和
  rebuild.get_conn_cursor()
 
  ##########################################################################
  ## 修改商品ID
  ##########################################################################
  # 获得商品的游标
  goods_sql = '''
    SELECT goods_id FROM goods
  '''
  goods_iter = rebuild.execute_select_sql([goods_sql])
  # 根据获得的商品ID更新商品表(goods)和订单商品表(order_goods)的商品ID 
  for goods in goods_iter:
    for (goods_id, ) in goods:
      rebuild.update_table_id('goods', 'goods_id', goods_id)
      rebuild.update_table_id('order_goods', 'goods_id', goods_id, rebuild.get_current_guid())
    rebuild.commit()
 
  ##########################################################################
  ## 修改订单ID, 这边我们规定出售订单ID和购买订单ID相等
  ##########################################################################
  # 获得订单的游标
  orders_sql = '''
    SELECT sell_order_id FROM sell_order_1
  '''
  sell_order_iter = rebuild.execute_select_sql([orders_sql])
  # 根据出售订单修改 出售订单(sell_order_1)、购买订单(buy_order_1)、订单商品(order_goods)的出售订单ID
  for sell_order_1 in sell_order_iter:
    for (sell_order_id, ) in sell_order_1:
      rebuild.update_table_id('sell_order_1', 'sell_order_id', sell_order_id)
      rebuild.update_table_id('buy_order_1', 'buy_order_id', sell_order_id, rebuild.get_current_guid())
      rebuild.update_table_id('order_goods', 'sell_order_id', sell_order_id, rebuild.get_current_guid())
    rebuild.commit()
 
  ##########################################################################
  ## 修改订单商品表ID
  ##########################################################################
  # 获得订单商品的游标
  order_goods_sql = '''
    SELECT order_goods_id FROM order_goods
  '''
  order_goods_iter = rebuild.execute_select_sql([order_goods_sql])
  for order_goods in order_goods_iter:
    for (order_goods_id, ) in order_goods:
      rebuild.update_table_id('order_goods', 'order_goods_id', order_goods_id)
    rebuild.commit()
  # 关闭游标
  rebuild.close_cursor('select')
  rebuild.close_cursor('dml')
  # 关闭连接
  rebuild.close_conn()
{% endhighlight %}

**完整的python程序：**[rebuild_id.py](http://www.ttlsa.com/wp-content/uploads/2016/02/rebuild_id.py_.zip)  
**执行程序**   

```python
python rebuild_id.py
```

* **最后查看表的结果**

{% highlight SQL %}
SELECT * FROM goods LIMIT 0, 1;
+---------------------+------------+---------+----------+
| goods_id            | goods_name | price   | store_id |
+---------------------+------------+---------+----------+
| 3791337987775664129 | goods1     | 9369.00 |        1 |
+---------------------+------------+---------+----------+
SELECT * FROM sell_order_1 LIMIT 0, 1;
+---------------------+---------------+---------+---------+--------+
| sell_order_id       | user_guide_id | user_id | price   | status |
+---------------------+---------------+---------+---------+--------+
| 3791337998693437441 |             1 |      10 | 5320.00 |      1 |
+---------------------+---------------+---------+---------+--------+
SELECT * FROM buy_order_1 LIMIT 0, 1;
+---------------------+---------+---------------+
| buy_order_id        | user_id | user_guide_id |
+---------------------+---------+---------------+
| 3791337998693437441 |      10 |             1 |
+---------------------+---------+---------------+
SELECT * FROM order_goods LIMIT 0, 1;
+---------------------+---------------------+---------------------+---------------+---------+------+
| order_goods_id      | sell_order_id       | goods_id            | user_guide_id | price   | num  |
+---------------------+---------------------+---------------------+---------------+---------+------+
| 3792076554839789569 | 3792076377064214529 | 3792076372429508609 |             1 | 9744.00 |    2 |
+---------------------+---------------------+---------------------+---------------+---------+------+
{% endhighlight %}

**建议：**如果在生产上有使用到snowflake请务必要弄一个高可用防止单点故障，具体策略看你们自己定啦。  

## 二、java版本

**代码一**

{% highlight java %}
/**
 * @author zhujuan
 * From: https://github.com/twitter/snowflake
 * An object that generates IDs.
 * This is broken into a separate class in case
 * we ever want to support multiple worker threads
 * per process
 */
public class IdWorker {
     
    protected static final Logger LOG = LoggerFactory.getLogger(IdWorker.class);
     
    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
 
    private long twepoch = 1288834974657L;
 
    private long workerIdBits = 5L;
    private long datacenterIdBits = 5L;
    private long maxWorkerId = -1L ^ (-1L << workerIdBits);
    private long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    private long sequenceBits = 12L;
 
    private long workerIdShift = sequenceBits;
    private long datacenterIdShift = sequenceBits + workerIdBits;
    private long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private long sequenceMask = -1L ^ (-1L << sequenceBits);
 
    private long lastTimestamp = -1L;
 
    public IdWorker(long workerId, long datacenterId) {
        // sanity check for workerId
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDatacenterId || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }
        this.workerId = workerId;
        this.datacenterId = datacenterId;
        LOG.info(String.format("worker starting. timestamp left shift %d, datacenter id bits %d, worker id bits %d, sequence bits %d, workerid %d", timestampLeftShift, datacenterIdBits, workerIdBits, sequenceBits, workerId));
    }
 
    public synchronized long nextId() {
        long timestamp = timeGen();
 
        if (timestamp < lastTimestamp) {
            LOG.error(String.format("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp));
            throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
 
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }
 
        lastTimestamp = timestamp;
 
        return ((timestamp - twepoch) << timestampLeftShift) | (datacenterId << datacenterIdShift) | (workerId << workerIdShift) | sequence;
    }
 
    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }
 
    protected long timeGen() {
        return System.currentTimeMillis();
    }
}
{% endhighlight %}

**代码二**

{% highlight java %}
public class IdWorkerTest {
 
    static class IdWorkThread implements Runnable {
        private Set<Long> set;
        private IdWorker idWorker;
 
        public IdWorkThread(Set<Long> set, IdWorker idWorker) {
            this.set = set;
            this.idWorker = idWorker;
        }
 
        @Override
        public void run() {
            while (true) {
                long id = idWorker.nextId();
                if (!set.add(id)) {
                    System.out.println("duplicate:" + id);
                }
            }
        }
    }
 
    public static void main(String[] args) {
        Set<Long> set = new HashSet<Long>();
        final IdWorker idWorker1 = new IdWorker(0, 0);
        final IdWorker idWorker2 = new IdWorker(1, 0);
        Thread t1 = new Thread(new IdWorkThread(set, idWorker1));
        Thread t2 = new Thread(new IdWorkThread(set, idWorker2));
        t1.setDaemon(true);
        t2.setDaemon(true);
        t1.start();
        t2.start();
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}