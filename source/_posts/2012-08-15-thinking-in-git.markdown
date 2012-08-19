---
layout: post
title: "Thinking in git"
date: 2012-08-15 23:57
comments: true
categories: [git,vcs,dag,architecture]
---

#  前言
最近工作中用到git,本着遇到什么学什么的精神,最近看了看git相关的资料和一些blog,将一些学习方法和结论总结一下.



###  头疼的命令	
在初次使用git时,我感觉这东西是个不太易用的东东,概念复杂,命令复杂(这还是被封装过的,用底层命令完全得理解整个系统的来龙去脉才有可能).  



很多blog会介绍一些fast food,比如本地仓库和远程仓库diff,然后贴个命令上去....有些时候命令未必靠谱(原因是git命令复杂,贴出来的命令未必真适应你的场景),最后搞的我只有祭出GUI来完成日常工作,实在是悲催.  
  
  	
####  方法 双管齐下!
工作中的常用命令通过查blog,man doc学习,工作时不拘泥于命令的复杂使用,而是快速学会常用命令,了解基本概念(个人觉得git社区版介绍不错[git社区版](http://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0CFcQFjAB&url=http%3A%2F%2Fgitbook.liuhui998.com%2F&ei=SsorUIvVFoyRiQel4YDoDA&usg=AFQjCNHVw05olZNWF6IDNhFykqVQNkN5wg),git pro入门看感觉一般).  



  
对于git这么久经考验的开源系统而言,不了解底层原理,和其他vcs产品的异同,设计的理念,发展过程,产品现状等等,那就算是白用了,而且芥末优秀的产品对于学习产品架构设计很有借鉴意义,对于此我个人推荐[The Architecture of Open Source Applications volume II](http://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0CFoQFjAB&url=http%3A%2F%2Fishare.iask.sina.com.cn%2Ff%2F23222298.html&ei=WMsrULvDB6aciAevuoCgDg&usg=AFQjCNGecM96Vuaa7rQGtyyGzlLNvk3hYw)  
  
  

------

先写到这,后续补充
