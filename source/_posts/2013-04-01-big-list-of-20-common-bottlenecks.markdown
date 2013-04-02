---
layout: post
title: "Big List Of 20 Common Bottlenecks"
date: 2013-04-01 20:04
comments: true
categories: [翻译,performance]
---

## tips
翻译水平有待提高.部分内容添加了个人的理解.

## 前言
在[Zen And The Art Of Scaling - A Koan And Epigram Approach](http://highscalability.com/blog/2012/2/27/zen-and-the-art-of-scaling-a-koan-and-epigram-approach.html)这篇文章中,Russell Sullivan提出了一个有趣的猜想:存在20个经典的瓶颈.这听起来疑似那个"只有20个基本的故事情节"的想法.这完全取决于你看瓶颈的视角,也许这个想法是对的,但实践过程中我们都知道瓶颈来自于无穷的"味道",所有的酸和灰.

- chunkify这个单词翻译的太纠结了,没明白啥意思,先意会一下吧.
- 酸我能理解是个味道,灰是啥.......
- sullivan在分布式数据库方面很牛,Alchemy Database没怎么研究过呢还,但是卖点是混合型数据库,貌似是redis+server side script实现,参考[zen and the art of scaling](http://highscalability.com/blog/2012/2/27/zen-and-the-art-of-scaling-a-koan-and-epigram-approach.html)

有一天[Aurelien](http://jsoftbiz.wordpress.com/)发送了他的瓶颈列表给我, 我们cc给了Russell,他给了我他的列表,我也有一份列表,于是产生了一份石头汤

- [石头汤](http://en.wikipedia.org/wiki/Stone_Soup)

Russell说这是他的"我希望在我年轻的时候我知道"列表,我想应该用一种不断充实的方式来看待它:拥有更多的经验,处理更多不同类型的项目,会有更多的教训你能够添加到这份列表中.当你读到这份列表,并且制作自己的列表时,你正在通过多年积累的经验和一点点挫折而不断变强,并且其中每一条都有一个故事值得grokking.

- grokking 实在是无法用一个词汇来描述....

## 瓶颈的个人理解

### Database:
- Working size exceeds available RAM

  工作内存超过了系统可用的内存,可能会大量出现malloc失败或者触发swap区不断的换入换出,或者触发内存碎片整理这种成本较高的动作.对于数据库来说,性能应该会下降的很快,这个不用说,内存总是比硬盘快.基本上离挂不远.

- Long & short running queries

  耗时较长的查询会占用系统资源一段较长的时间无法释放,会带来性能的波动.通常来说,业务上对于数据查询的场景是属于OLTP or OLAP需要分析清楚先,并以此作为技术选型的评估因素之一.

- Write-write conflicts
  
  大量的并发写冲突问题,此问题我分为两种情况:对系统整体的影响,热点的问题.
  
  - 整体的影响如何解决?
	
	Sharding strategies often involve two techniques: partitioning and replication. With partitioning, the data is divided into small chunks and stored across many computers. Each of these chunks is small enough that the computer that stores it can efficiently manipulate and query the data. With the other technique of replication, multiple copies of the data are stored across several machines. Since each copy runs on its own machine and can respond to queries, the system can efficiently respond to tons of queries for the same data by adding more copies. Replication also makes the system resilient to failure because if any one copy is broken or corrupt, the system can use another copy for the same task.
  
  - 热点

	scale up性能,比如数据库热点数据,可以单独拿出来提供服务.热点操作的针对性优化,比如对于计数器场景,cpu做必要的优化,写可以用thread local or array + padding.  

- Large joins taking up memory
  
  尽量避免两张大表join,将sql拆开为两次简单查询是一种解救办法,但可能会出现too much short running queries问题,需要结合性能数据分析权衡ROI
.

### Virtualisation:
这个领域...比较小白啊,先放着

- Sharing a HDD, disk seek death
  
- Network I/O fluctuations in the cloud

### Programming:

- Threads: deadlocks, heavyweight as compared to events, debugging, non-linear scalability, etc...
  
- Event driven programming: callback complexity, how-to-store-state-in-function-calls, etc...
  
  callback的复杂性:callback有两种,block callback&async callback,通常这种模式提供给模块使用者,来扩展已有功能的一种方式.我个人感觉上callback对于一些严谨的模块维护者来说,侵入性较强,难以控制.模块使用者会比较容易写出破坏接口语义的实现,特别是对于async的接口.
  
  store state...不知所云

- Lack of profiling, lack of tracing, lack of logging
  
  使用成熟的工具,提供profiling是必须的.
  跟踪---分布式环境跟踪是个问题,事件需要统一收集和分析,etc..kafka
  logging --例子很多了.不用一台台scp download日志,否则会疯了.

- One piece can't scale, SPOF, non horizontally scalable, etc...
  
  可扩展性是架构设计需要解决的基本问题,但也是非常复杂的问题,无状态,分区可扩展性.
- Stateful apps
  
  有状态就可能涉及到状态的持久化问题,同时状态的可靠性持久化需要一些复杂的的机制来保证:write ahead logging, quorum, replication,raid,paxos etc...

- Bad design : The developers create an app which runs fine on their computer. The app goes into production, and runs fine, with a couple of users. Months/Years later, the application can't run with thousands of users and needs to be totally re  architectured and rewritten.

  这事天天发生
  
- Algorithm complexity
  
  复杂意味着维护性问题,可扩展性问题,调优问题.
  
- Dependent services like DNS lookups and whatever else you may block on.
  
  前端性能优化...域名可以拆分,但不能过多.
  
- Stack space
  
  栈还是有开销的....特别是对于线程创建后的栈固定分配时,线程创建的开销还是不可忽视的,虽然有所谓的copy on write优化...
  无其他想法了,除了line优化降低栈深度.
  
以后在补充吧.
