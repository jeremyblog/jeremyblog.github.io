---
layout: post
title:  持续大量的nack造成内存爆掉问题追踪
---

### 背景知识
Rabbit MQ在其官网tutorial的第二章[Message acknowledgment](http://www.rabbitmq.com/tutorials/tutorial-two-python.html)段节中指出，为了保证消息的永不丢失，Rabbit Mq提供了消息的ack机制:服务器仅在收到客户端(消费者)回传的ack报文后才会将消息释放和删除。
<br>
期间如果消费端被关闭或者出现宕机，此时因为没有回传ack，所以服务端认为消息没有被正确处理，从而将其重新返回队列中，以供其它的消费者使用。

### 问题描述
realtime.index的项目采用上述的ack机制来保证数据的不丢失。

		if (solr4jServerManager.getInit().get() == false) {
			setSuccess(false);
		} else {
			setSuccess(//业务逻辑处理结果);
		}

在程序没有准备好或者处理失败的时候，回复nack可以避免阻塞队列，再次收到消息的时间非常短。
<br>
以上的代码在测试环境和kdtest环境中均正常，但是在正式环境就出现问题了。
<br>
由于代码部署后业务逻辑处理结果一直是失败的，所以程序实际上是不断地执行else的逻辑，一直给Rabit mq回复nack报文，又持续不断地从rabbit mq消费数据。
所以这时候的队列虽然有消费者，但是因为每次都不成功，所以队消息的大小是不断增长的。
<br>
持续10分钟后，观察队列的消息，此时队列大小已经达到6k，虽然在队列界面显示占用的内存还非常小，但是整体的内存开始出现快速增长的现象，以十M甚至百M的单元跳动，并很快在不到5分钟内，Rabbit mq就触及预警线。
<br>

### 原因分析
我们是通过排查确认的问题，但是并没有从rabbit日志中得到真实的原因，阅读官网提报的问题清单，两个案例和我们比较类似。
<br>
一个是以优化的方式提交给作者的 [Prevent consume message NACK loops (DoS)](https://github.com/rabbitmq/rabbitmq-server/issues/1020)
> A customer recently got into a busy loop whereby they were consuming thousands of messages off the queue, then NACK'ing them all, then consuming the queue again, NACK'ing again, and so forth. We noticed an extremely high level of CPU consumption as a result.

<br>
一项是求助文章[is peeking at the queue dangerous by disabling auto-ACK and getting all data
](https://groups.google.com/forum/#!msg/rabbitmq-users/KxEnOH-oT9g/rMXzeM1KFIcJ)

<br>
从文章可以梳理出这样的结论:
<br>
这种loop nack的方式在队列消息比较多(超过几k)，且qos比较大的情况下容易出现短时间内CPU和内存拔高的问题。从rabbit mq的维护者提供的思路来看，造成种问题的原因是rabbit mq收到了nack报文后，会把这个失败的报文放到队列的后面，以实现重新redelivered。
<br>
这样带来的问题有两个:

1. 一是本来应该释放掉的内存没有释放

2. 另外就是把消息重新加入队列的时候，rabbit mq可能会加大的队列长度(和gc不及时有关)。

也就是说，为什么在测试环境没有问题，一是队列中消息比较少，二是消费的QOS设置也比较小。如果不影响gc，内存都能自动恢复到一个正常值。

### 关键数据对比
<table>
  <thead>
    <tr>
      <th><p>环境</p></th>
      <th><p>QOS</p></th>
      <th><p>数据per second</p></th>
	  <th><p>节点数</p></th>
	  <th><p>内存大小</p></th>
	  <th><p>报文大小</p></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><p>dev</p></td>
      <td><p>10</p></td>
      <td><p>3小时不到1k</p></td>
      <td><p>1</p></td>
      <td><p>不到1g</p></td>
      <td><p>1K</p></td>
    </tr>
    <tr>
      <td><p>pro</p></td>
      <td><p>100</p></td>
      <td><p>3小时大约4w</p></td>
      <td><p>2</p></td>
      <td><p>分别为5G和6g</p></td>
      <td><p>1K</p></td>
    </tr>
  </tbody>
</table>

### 编码建议
以上的问题，作者并不认为是一个bug，并不会特别优化这一个，而是给出了两条编码建议
> - Queue length of message TTL (both already available)
> - The limit on how many times a message can be requeued

<br>
这个原则就是建议采用ttl和Dead Letter两种方式来避免它。TTL就是设置消息的存活时间，而Dead letter则在消息超过存活时间后将消息转为到一个独有的exchange上。
<br>
这个独有的独有的exchange以及它的queue，和我们平常普通的exchange，queue是一样一样的。只是设置它们的绑定需要 使用 **x-dead-letter-exchange** 参数，同时，也可以指定路由键，通过 **x-dead-letter-routing-key **参数。
<br>
以下是这两种方式的数据流转
![编码建议](https://i.stack.imgur.com/kLx7a.png)

### 代码示例
	@Bean
	public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
	  return new Jackson2JsonMessageConverter();
	}

	@Bean
	public Queue workQueue() {
	  Map<String, Object> args = new HashMap<String, Object>();
	  // The default exchange
	  args.put("x-dead-letter-exchange", "");
	  // Route to the incoming queue when the TTL occurs
	  args.put("x-dead-letter-routing-key", RETRY_QUEUE);
	  // TTL 5 seconds
	  args.put("x-message-ttl", 5000);
	  return new Queue(WORK_QUEUE, false, false, false, args);
	}
	
	@Bean
	public RabbitTemplate outgoingSender() {
	  RabbitTemplate rabbitTemplate = new RabbitTemplate(cachingConnectionFactory);
	  rabbitTemplate.setQueue(workQueue().getName());
	  rabbitTemplate.setRoutingKey(workQueue().getName());
	  rabbitTemplate.setMessageConverter(jackson2JsonMessageConverter());
	  return rabbitTemplate;
	}
	
	@Bean
	public Queue retryQueue() {
	  return new Queue(RETRY_QUEUE);
	}

### 参考资料
1.  [google rabbitmq-users论坛](https://groups.google.com/forum/#!searchin/rabbitmq-users/nack$20$20redelivery$20|sort:relevance/rabbitmq-users/KxEnOH-oT9g/rMXzeM1KFIcJ)
2.  [RabbitMQ consumer的一些坑](http://www.itdadao.com/articles/c15a1042895p0.html)
3.  [RabbitMQ 官网tutorials](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)
4.  [RabbitMQ github issues](https://github.com/rabbitmq/rabbitmq-server/issues/1020)
5.  [stackoverflow 关于这个问题的讨论](https://stackoverflow.com/questions/28604332/rabbitmq-ack-or-nack-leaving-messages-on-the-queue)
6.  [Acknowledging a Message Using RabbitMQ](http://pmichaels.net/2016/10/28/acknowledging-a-messing-using-rabbitmq/e)
7.  [Dead Letter Exchanges](https://www.rabbitmq.com/dlx.html)
8.  [Spring, RabbitMQ & Dead Letter Exchanges](https://www.sourceclear.com/blog/Spring-RabbitMQ--Dead-Letter-Exchanges/)