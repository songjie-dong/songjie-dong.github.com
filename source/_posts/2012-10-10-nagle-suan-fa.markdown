---
layout: post
title: "nagle 算法"
date: 2012-10-10 16:55
comments: true
categories: [TCP,performance,network,翻译,algorithm]
---

在各种场合看到涉及网络通讯的代码时,总会到到TCP_NODELAY的设置,也就是绕过nagle算法,虽然了解这个算法的基本原理,但一直没有机会追根述源,看wikipedia上关于nagle算法的介绍,很详细,所以也懒得分析,直接简单翻译并记录一下该算法,以免后续总问自己这算法到底能功能用在哪?

## 介绍
nagle算法以其发明者john nagle命名,该算法通过减少在网络上传输的包来改进网络上传输的效率.

nagle在文档:Congestion Control in IP/TCP Internetworks中描述了所谓的"small packet problem", 即当一个应用重复发送小块数据,通常大小只有1个byte.TCP的包体中有40byte的header(20byte的TCP信息,20byte的ipv4信息),而这会导致41个byte的包只有一个byte的有用信息,这是一个巨大的开销.这种情况常常出现在使用telnet会话时大部分键盘输入会生成一个byte有效数据并被立即传输时,更坏的是在慢速网络环境下,大量这样的包会在同一时间传输,会潜在的导致拥塞崩溃.

## pseudocode

{% codeblock  lang:java %}
	if there is new data to send
	  if the window size >= MSS and available data is >= MSS
		send complete MSS segment now
	  else
		if there is unconfirmed data still in the pipe
		  enqueue data in the buffer until an acknowledge is received
		else
		  send data immediately
		end if
	  end if
	end if	  
{% endcodeblock %}

## 问题
80年代初,有个组织提交了名为TCP Delayed Acknowledgment的算法,该算法会合并多个ack包以达到提升效率的作用,最大的ack延迟为500ms.当两个算法同时启用时,出现write-write-read的顺序时则由于第二个写可能的延迟导致read时间过长.由于这个原因,TCP实现提供了一些应用接口来关闭nagle算法,通常这个可选项名为TCP_NODELAY.(关于write-write-read带来的问题还有些不是很清晰)

nagle算法适用于任何大小的数据,如果一次写请求需要发送的包跨度为 2n packets,则最后的数据包会被buffer,直到收到前一个包的ack消息,延迟会增加上百ms,而且是由于这个算法导致了额外的开销.如果数据大于一个packet时,requester或responser最好都关闭该算法.




