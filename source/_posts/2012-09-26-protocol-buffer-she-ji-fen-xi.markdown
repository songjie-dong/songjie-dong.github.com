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
想到俺也曾经在一个子系统里整过n种协议的RPC,很痛苦,健壮的协议可以降低复杂系统依赖带来的巨大成本,比如某核心系统升级接口,发现以前DIY的二进制协议难以做到向下兼容,于是乎,得整出两套接口,然后让依赖系统缓慢升级,这个过程需要很多人肉的工作,开发成本,运维成本,安全成本,百人以上的技术团队如果经常有这样的升级,显然开发人员的幸福感会下降的很快.
### Hello world
{% codeblock example message lang:java %}
  message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phone = 4;
}
{% endcodeblock %}

一些对分析设计有用的信息,field的类型为协议的兼容性提供了参考,数据类型是平台无关的,message可以组合和继承,数据类型丰富,message的描述能力强,高级功能忽略先.

### 设计关注点
#### Encoding ####
message的编码方式决定了传输的数据大小,传输性能,还有decoding的性能,同时还要兼具扩展性,pb的编码对于数字类型做了很多的优化,基于[BaseVarints](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN ) 这种变长编码方式对整型数据编码,使用了标志位来标识传输数据单元的长度,对于小于2^28 的数据来说,这种方式可以减少冗余传输量,对于大于2^28 的数据最好用fixed类型,否则会多出一个byte.对于负数的优化更难以理解一点,对于varint来说,正数,特别是小正数,可以节省很多的空间,但是对于负数则不然,因为高位总是有标志位,所以必须至少按5个字节来存,但考虑到兼容性,比如int32<->int64之间的转换问题,则总是使用10个字节来varint的存储负数 [why nagetive int always use 10 bytes](https://groups.google.com/forum/?fromgroups=#!topic/protobuf/fU0SVchScA0 ),官方推荐使用sint这种类型,使用了zigzag这种方式来编码,基本思路是把负数变正数,这样就又可以发挥变长的优势了.基本上协议体内部的数字都是变长的.协议中使用了极少的协议体信息,相对与xml和json,很多元信息都在message中,只要通讯双方有兼容的message即可.这些二进制流在被读取时可以简单的通过一些位运算即可decoding,so性能自然在xml和json之上了. 

#### Message Extensible ####
message定义本身具备较强的扩展性,但要注意类型和field type,否则会带来兼容性问题.

#### Multi-Platform ####
统一使用了LE编码顺序,message数据类型与平台无关,encoding与平台无关,提供了编译器,针对不同的语言生成stub,跨平台,隐藏细节.
#### Compatibility ####
message数据类型在encoding过程中考虑了一些兼容性的问题,在对message升级的过程中,常见情况下都是可以满足需求的,但还得好好研究下官方对于message升级的说明,否则可能带来一些困惑.对于持久化的数据来说更要小心.其实在开始设计message的时候应该合理和小心的使用数据类型和字段类型,pb提供了不错的兼容性而无需我们而外coding,但不代表我们可以完全依赖它,总有些情况是无法满足的,所以在设计初期应该考虑到.
### 应用场景

1.   复杂分布式环境,良好的兼容性和扩展性,平台无关.
2.   小数据通讯,存储,对于kb级一下的数据,性能较好,但不为大数据准备,如有需要请合理将大数据分片.

### 问题

1.  小数据通讯
2.  处于发展中,只使用必要功能,除非你深入研究原理
3.  管理好messag

----------

## 总结
设计总是要把问题规约到某几个核心问题上,放弃华而不实的功能点,放弃建造巴别塔.以核心问题为标准来衡量设计的必要性和设计的关注点.

## 参考

- [varint的一些编码说明](http://www.cppblog.com/true/archive/2009/09/11/95873.html )
- [官方说明,注意关于负数的那段](https://developers.google.com/protocol-buffers/docs/encoding?hl=zh-CN )
- [很全面的介绍](http://www.ibm.com/developerworks/cn/linux/l-cn-gpb/?ca=drs-tp4608 ) 
- [完全做到动态message,当然有代价,是否合理不作讨论](http://www.searchtb.com/2012/09/protocol-buffers.html) 

