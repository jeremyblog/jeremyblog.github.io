---
title: 运维的一些坑
---

# 运维的一些坑
{:.no_toc title="运维的一些坑"}

这个章节中级的读者，需要一定使用zookeeper的经验。

zookeeper的出现让我们从复杂的分布代码中解放出来，让我们轻易地就可以实现并发锁，轻松地就能获得强一致性的数据，并且从此不再采用耗资源轮询方式，而是使用watch获取数据变更的通知。它让我们的编码更简单了，可是它不是银弹，它毫无例外地遵循了解决一个问题必定带来另外一个问题的规则。

所以~~有些坑我们不得不防。

现在已经不是一两台机器就能胜任的时代了，大多公司的面临的问题远比过去复杂得多，对于新工具和技术的使用面临的挑战和风险也大得多。而有些坑(特别是近在高并发高负载等极端情况才会出现问题)，越早了解就越能帮助你早日从泥潭中脱身。

### 细数zookeeper的坑
个人遇到的一些坑：
*	几十万的节点，加上几十万的watch的场景，(这个从少量的大型互联网分享中得知)问题在于超多节点带来了大量的链接和流量，特别是节点变化时，众多的watch带来的流量是非常大的。这不仅对zookeeper本身有影响，对整个网络的架构都带来很严重的影响。
*	full gc时间过长，造成session过期出现的假死情况
*	JVM 内存设置太小，频繁swap严重影响zookeeper的性能，因为zookeeper的所有数据都是在内存中的。
*	频繁的写日志，默认zookeeper定期需要dump上述内存快照到磁盘，在高峰期可能带来问题(尤其是虚拟机的IO大多不会太好)。
*	有着大量请求的细粒度锁，比如抢红包。
*	客户端使用10.62.x.X:2181，10.63.x.x这样的IP串连接，后续想添加或者修改服务器不好实现。
*	脑裂现象，分区容灾性差
*	zookeeper的集群节点不能是偶数

### 追源溯本，zookeeper的一些原理
~~~
1.	zookeeper的投票，投票的节点数一定要是奇数，因为zookooper采取了大多数的原则
2.	一个写操作需要半数以上的节点ack后才得以提交，所以集群的节点越多可用性越强，但是TPS肯定会受影响
3.	zookeeper的所有节点及节点数据都会放到内存里(树状结构)，并定时dump到磁盘。
4.	为保证数据的一致性，zookeeper每次都会写txlog，同时为了避免日志文件的变大，它还提供了定时清理日志的机制
5.	client和zookeeper之间为了维持长连接，采用了心跳的机制，在session超时时间内如果收不到心跳，则该session会过期。
6.	client可以通过watch zookeeper的节点有数据
~~~

### 知行合一，如何打造zookeeper
首先避免手中有锤子，到处是钉子的心态，扬长避短才是我们的目标。
1.	最小的集群
采用奇数的节点数，基于可靠的数据，100台机器中，一年有且仅有1到2台机器的故障率，个人觉得采用至少5个节点就非常安全了。
2.	隔离集群
按角色，把leader+follower与observer分开，仅让observer对client服务(类型与读写分离，leader+follower实现写，observer读)；按业务员分(垂直)，比如有服务发现，消息，定时任务等业务，某个业务建立独立的集群。
3.	版本，避免脑裂现象
脑裂现象还是比较常见的，full gc过长，时钟等等造成假死的情况都会发生，为避免脏数据的情况，必须使用版本提交数据。
4.	手动清理数据
建议关闭autopurge.purgeInterval=0,然后使用crontab的机制在业务低谷的时候清理数据。
5.	JVM设置，避免swap
zookeeper启动时会加载conf文件夹下的配置文件
#!/usr/bin/env bash
JAVA_HOME = #java home
ZOO_LOG_DIR= #log dir
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
VJMFLAGS="..."
6.	IP地址
对于客户端的配置，建议采用集中的配置管理，这样客户端就不需要再关注它了
对于集群的配置，结合hosts，
hosts:
192.168.1.1 zk1
192.168.1.2 zk2
配置:
server.1 = zk1:2081
server.2 = zk2.2081
如果有变更，仅需要更改hosts即可
~~~
java如何刷新DNS CACHE？
默认java不会刷新dns的，采用hosts的方式修改了配置需要重启zookper，这个在生产环境很难接受。幸亏有一个方案
修改$JAVA_HOME/jre/lib/sesurity/java.security，将networkaddress.cache.ttl=60
~~~
7.	避免羊群效应
假设这样的场景，为了实现分布式的锁，某个节点有大量watcher(参考状态事件一节)，在节点变化时需要发送大量的通知，且有大量的请求过来取数据。解决方法就是：客户端创建/lock/lock-的有序节点，假定我们的列表已经按需要从小到大排序，默认最小的节点优先拿到锁，然后次小的节点监听拥有锁的节点，依次下去。
8.	低版本zk client的坑
有大量watch的客户端，在重连的时候需要将所有的watch都订阅一遍，而zookpper对单个数据包有1M的限制，所以出现了重复连接又连不上的问题。
9.	强依赖变为弱依赖
采用守护进程和zookeeper连接，当监听到变化的时候，守护进程在将变化通知到客户端，这样可以大大降低了zookper的连接数量；同时守护进程和客户端都应该zookeeper数据拥有缓存，在zookeeper不可用的时候，应该不影响整体业务的使用。