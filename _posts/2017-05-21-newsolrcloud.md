---
layout: post
title: zookeeper在solrcloud集群的引用
---
solr1.4之都是基于复制的模式还实现集群的，这个模式的特点是非常简单，一个write，然后n个读的采用树状的结构进行复制，用户优先访问子节点读取数据。
<br>
这个设计的可以避免所有的节点同时从主节点复制数据，出现雪崩现象。
<br>
但是由于"复制"是基于http进行sync，且比较的是segmengt的版本。solr默认保留10个segments，超过时会进行合并与优化，导致增量同步变成了全量同步。如果不进行优化，数据量非常大的时候，solr的查询会比较缓慢。
![](http://oq6i0apwz.bkt.clouddn.com/tree1.png)


**solrcloud**

solr 4.*版本以后，基本上建议是用solrcloud代替复制模式了。
![](http://people.apache.org/~markrmiller/2shard4serverFull.jpg)

新docment进来的时候，solr通过zookeeper和一致性hash得知消息的shard和leader()，然后把消息转发至目标的leader。
<br>
举例来说当Solr Shard建立时候，Solr会给每一个shard分配32bit的hash值的区间，例如两个shard分别为A,B，A的hash值区间为80000000-ffffffff，B的hash值区间0-7fffffff。hash策略会根据document ID计算出唯一的Hash值，并判断该值在哪个shard的hash区间内。
<br>
<br>
而在消息发送至shard时，为提高并发能力用户并不需要知道哪个是leader，而是随意提交给shard任意节点，这时候的路由如下:
<br>
首先如果它不是leader，solr会把请求转给和自己同Shard的Leader。
<br>
接着Leader把文档路由给本Shard的每个Replica，直到收到所有replica的回复成功后才回复正确。
<br>
特别地，如果文档基于路由规则(如取hash值)并不属于当前的Shard，leader会把它转交给对应Shard的Leader,再由对应Leader会把文档路由给本Shard的每个Replica。
<br>


**ulog 失败节点如何恢复？**

Leader接受到update请求后，先将update信息存放到本地的update log，同时会给document分配新的version，然后发送至replica。
<br>
而replica收到请求后，同样先写入ulog，最后再写入到本地。
<br>
SolrCloud并不会关注那些已经下线的replica，而是在上线时候recovery进程通过ulog进行恢复。

**solrcloud的缺点**
为实现高并发的性能，solrcloud采用的由leader转发的模式，由于zookeeper session超时其实有一些延迟(默认为6s)，这期间的时候可以考重试来避免。
<br>
但是如果leader收到消息转发时出现故障，随后如果还不恢复就会开始选举新的leader，导致数据丢失。
<br>
可见solrcloud并不保证数据的一致性。

**自定义solr集群**

一、通过zookeeper 来发现活跃的shard，这里取消了leader，leader被提升为realtime.index的服务器
<br>
二、realtime.index收到新的docment后，直接提到到MQ队列中(持久化)，并设为主动ack模式，这样可以保证写入成功后才从MQ中移除。
<br>
三、realtime.index 写入的时候基于最终一致性的原则，写入时采用异步的方式，收到写入结果后决定是从MQ中移除还是重新写入。
<br>
四、realtime.index写入的前会写日志到一个mongo的集合中，这是一个固定大小的set，可以记录10w笔历史数据的更新。并同时为每个写入set的数据提供一个版本，写入成功的时候维护一个set:version列表。每次solr启动进行注册时，realtime.index会收到一个watcher事件。然后比较版本，如果版本并一致:
<br>
<br>
1 版本是最新的不需更新
<br>
2 版本不最新的，且seq在日志的seq范围之内，也就是必有seq+1 = indexLog.getReq()，直接从日志恢复
<br>
3 新加入的机器或者无法通过日志来完成同步，必须进行全量更新，直接使用solr的复制模式，不过需要记录期间的增量更新。
<br>
 
这个模式可以保证最终数据的一致性，缺点就是会出现部分用户拿到的数据版本不一致，而且会出现进行全量复制的结果。
<br>
**优点**是简单，不会丢失数据，实现逻辑也非常简单。
![](http://oq6i0apwz.bkt.clouddn.com/newsolr1.png)

