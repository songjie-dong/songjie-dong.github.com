---
layout: post
title: "Is parallel programming hard? - 2 "
date: 2012-11-15 14:39
comments: true
categories: [cas]
---

## 问题
CAS是否是绝对的灵丹妙药呢?

## 测试
根据并发编程实践中的结论,极端的竞争条件下,Lock表现出比CAS更好的性能,在中等的竞争条件下,CAS明显胜出.

## Is parallel programming hard 书中关于CAS过程的分析
对于一次CAS操作,如果是最好的情况:当前处理的core的l1 cache line 命中,则直接操作,如果miss,则会通过Interconnect通讯其他core,寻找CAS操作的数据,core间传输数据的开销不低,在4-CPU 1.8GHz AMD Opteron 844 System上,整个miss的CAS操作需要500个clock cycle.而命中的情况下只需要63个clock cycle.

另外java里实现的CAS都有一定数量的spin操作,如果大量的竞争失败,对于CPU来说则是做了很多无用功.

## 结论
吞吐量的测试结果是在高并发的情况下:CAS的miss开销+CPU无用功 > Lock的开销(context switch, cache miss). 


