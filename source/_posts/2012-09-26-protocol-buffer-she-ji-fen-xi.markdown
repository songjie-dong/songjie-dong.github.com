---
layout: post
title: "protocol buffer 设计分析"
date: 2012-09-26 17:27
comments: true
categories: [protocol buffer,通讯协议,序列化,高性能,编码] 
---

## 诞生
传说google在索引系统的充斥着这样的代码
{% codeblock ugly code by google lang:java %}
	if (version == 3) {
		...
	} else if (version > 4) {
		if (version == 5) {
			...
		}
		...
	}
{% endcodeblock %}
	
各种通讯协议的维护,版本兼容性,大型分布式系统升级维护成本,性能低下等等皆是让人头痛的问题,最终它们憋不住了,整了个protocol buffer(以下简称pb)出来,pb主要关注通讯协议本身的设计,充分考虑到了之前的各种问题,并将这些繁文缛节的东西考虑到了设计中.

----------

## 分析
### Hello world
### 设计关注点
#### Serialization ####
#### Encoding ####
#### Extensible ####
#### Multi-Platform ####
#### Compatibility ####
### 应用场景
### 问题


----------

## 总结
## 参考
