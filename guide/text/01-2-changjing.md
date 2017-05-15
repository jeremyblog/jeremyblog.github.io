---
title: Zookeeper的使用实例
description: 提供一些zookeeper的使用实例，以便直观地理解zookeeper的用处
keywords: zookeeper,实例
---

# Zookeeper的使用实例
{:.no_toc title="Zookeeper的使用实例"}

这个章节适合所有的读者

本章节延续应用场景一节，列举一些Zookeeper的使用实例，以便更直观地理解其用处。

**Apache HBase** 开源的数据存储仓库组件，这里zookeeper用于选举一个集群内的主节点，以跟踪可用的服务;同时也保存集群使用的元数据。

**Apache Kafka** kafka是一个基于订阅-发布(pub-sub)模型的消息系统，目前在已被国内知名的大厂应用。zookeeper用于检测奔溃，实现topic的发现，并持久化topic的生产和消费状态。

**Apache Solr** Solr是一个企业级的搜索平台，solr的分布式版本solrCloud，使用zookpper来存储集群的元数据，并保证这些元数据强一致性。

**Yahoo！ Fetching Service** 是爬虫实现的一部分，通过缓存内容的方式高效地获取网页信息，同时确保满足网页服务器的管理规则(例如robots.txt)。改服务采用zookeeper实现主节点选举、奔溃检测和元数据存储。

**FacebookMessges** 集成email、短信、Facebook聊天和Facebook收件箱等通信通道应用。zookeeper扮演控制器的角色，用于实现数据分片、故障恢复和服务发现等功能。

**Dubbo** 一个布式服务框架,致力于提供高性能和透明化的RPC远程服务调用方案,是阿里巴巴SOA服务化治理方案的核心框架。其使用zookeeper作为其主要的注册中心，实现服务的发布、管理等功能。

除此之外还有很多的应用，这里不一一详举，并发的开发往往极其复杂，现有的语言中，java、go都提供了很好的并发的工具，都是这并不意味着我们已经从并发的泥潭中脱身出来，而zookeeper的工作恰好是提供了一种简单的并发处理机制，它借鉴了之前的一些分布锁管理或者分布式数据库来实现系统间的协作，同时简化用户使用的复杂性，让我们可以专注业务开发。