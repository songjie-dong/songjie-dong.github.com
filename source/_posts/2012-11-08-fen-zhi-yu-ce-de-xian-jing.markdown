---
layout: post
title: "分支预测的迷思"
date: 2012-11-08 18:29
comments: true
categories: [branch predictor,分支预测,性能]
---

## What
首先,看看什么是分支预测,这个技术的出现是由于CPU技术的进步,现代CPU都采用了流水线结构,而且流水线越深(按深入理解计算机提到的是15),利用率越高,但流水线引入了一个问题,就是需要预先将指令取出并reordering再放入流水线,而程序中的分支则导致无法预取指令,必须等到evaluate条件指令之后才能确定,这会导致流水线无法充分利用.本文介绍一些基本的关于分支预测的概念和实现方式,具体的概念和资料请参考[分支预测器](http://zh.wikipedia.org/zh/%E5%88%86%E6%94%AF%E9%A0%90%E6%B8%AC%E5%99%A8 ) 

intel官方数据分支预测器能保证在工业测试中达到94%+的成功率,但这就像intel官方宣布的MIPS一样不靠谱,实际的情况有时会不如人意,分支预测成功率低会导致流水线需要重新取指并reordering,大约开销在20 clock cycle以内(数据各有差异,但我没看到20以上的),通常在5%以上就值得关注,10%以上就难以忍受了.

## 分支预测的主流技术

分支预测需要做两件事:
1. 预测branch可能的方向
2. 从BTB中找到对应的PC地址,取指并reording(实际情况步骤会更多)

BTB可以理解为一个hashmap,里面保存了某branch对应方向对应的需执行指令地址,BTB大小有限.

预测算法:

1. 饱和计数
   使用两个bit保存具备4种状态的状态机,状态机具有强不选择,弱不选择,弱选择,强选择,说简单点就是只有两次选择到同样的分支,状态机才会选择改变分支方向,早期的奔腾系列使用了这种方法,测试结果达到93.5%的准确率,但我们很容易就可以发现它的弱点.

2. 两级自适应预测器
   记录n次branch history的结果作为一个当前模式,同时存在一个pattern history table,这里面记录了n个长度模式和n个对应的饱和计数器.简而言之这是一个基于模式识别,具备很强的预测模式能力的预测器.
   通常来说,每个分支都需要一个独立的pattern history table,称之为本地分支预测,但考虑到对于m条分支预测指令,n次长度的历史模式来说,需要消耗的内存极大,所以有了全局分支预测,它提供了共享的pattern history table,好处是如果多条分支预测相同,则可以共享预测的效果,但如果多条分支只在部分情况下共享分支结果则适得其反.本地分支预测达到了97.1%的准确率,全局分支预测则为96.6%,当然我们需要费点力气才能发现它们的弱点.
   
3. 静态预测
   总是用一种方向来预测.编译器可以提供hint给cpu,适用类似if (xxx == null)的场景.
   
4. 循环分支预测
   对于现代cpu都会对loop的跳出条件做特殊优化,他会记录条件值和当前循环次数,当达到退出条件时直接预测跳出,这个计数值不会无限大,intel官方数据是16,也有说64的.
   
5. 融合分支预测
   本地和全局预测组合.

## 案例&分析

以下列举一些在intel core 2上的测试用例,用例只涉及到对某些具体模式的分支情况做讨论,不涉及实际应用中的复杂模式以及资源竞争的影响,同时结论也不一定适用于其他类型的CPU,这么做的原因在于:对分支预测来说,难以给出统一的结论性原则,但我们需要考虑如何识别潜在分支预测失败因子以及更重要的避免不必要的预测,或者消除它们.

数据采集使用perf这个linux下的系统性能调优工具,我们用该工具来获取分支预测数据,[介绍](http://www.ibm.com/developerworks/cn/linux/l-cn-perf1/index.html ),vtune是intel官方的分析工具,有空可以装来玩玩.

单表达式随机化:
{% codeblock single condition lang:java %}
  Random random = new Random(seed);
  for (int i = 0; i < 100000; i++) {
	  for (int j = 0; j < 64; j++) {
		  if (random.nextInt(2) == 1) {
			  //do something
		  } else {
			  //do something
		  }
	  }
  }
{% endcodeblock %}
这是一个最为简单,貌似看起来也很具有破坏性的例子,但是intel是强大的,根据一些分析,intel使用了多种预测器的组合,同时具备很强的自适应性,so结果很惊人:
branch-misses 2%-3% of all branches

结果基本在3%以内,也就是97%的准确率,由于intel对于分支预测并未公开过细节,所以不做过多讨论.

增加一个判断条件:
{% codeblock multi-condition lang:java %}
  Random random = new Random(seed);
  for (int i = 0; i < 100000; i++) {
	  for (int j = 0; j < 64; j++) {
		  if (random.nextInt(2) == 1 || random.nextInt(2) == 1) {
			  //do something
		  } else {
			  //do something
		  }
	  }
  }
{% endcodeblock %}

结果:
branch-misses 8%-12% of all branches

注意我的这个case,if判断内两个表达式都完全随机,两者毫无关联,此时miss率上升到8%-9%,随着我不断在if中加入随机性的表达式,miss率逐步提到12%左右.

失败模式:单if多随机表达式的杀伤力很强,消除方法可以考虑合并二者,比如上面的例子可以将表达式相加判断是否大于1,复杂情况考虑各种淫荡的位操作.

还有一种方式适用于表达式与loop无关的场景,可以将if外提,从而消除分支预测(编译优化).预测即使miss率很低也会消耗很多资源,而影响到其他的branch.

多条件分支:
{% codeblock if-else-condition lang:java %}
  Random random = new Random(seed);
  for (int i = 0; i < 100000; i++) {
	  for (int j = 0; j < 64; j++) {
		  if (random.nextInt(8) == 1) {
			  //do something
		  } else if (random.nextInt(8) == 2 {
			  //do something    ... next else if 
		  } else {
			  //do something
		  }
	  }
  }
{% endcodeblock %}
这种场景同样会带来5-8%左右的miss率,这个case中参数完全随机,现实中不太会出现这样的情况,这种模式在部分情况下可以用mapping的方式去消除.

循环退出条件的优化-循环展开:
{% codeblock before loop-unrolling lang:java %}
  for (int i = 0; i < 100000; i++) {
	  for (int j = 0; j < 65; j++) {
		  a[i][j] = j;
	  }
  }
{% endcodeblock %}
{% codeblock after loop-unrolling lang:java %}
  for (int i = 0; i < 100000; i++) {
	  for (int j = 0; j < 65; j+=2) {
		  a[i][j] = j;
		  a[i][j+1] = j+1;
	  }
  }
{% endcodeblock %}
将循环表达式中innerloop的循环次数降到64(需要考虑CPU的实现)以下,则可以使用CPU的循环分支预测器.
这种方式如果两层循环,且外循环很大的情况下容易出现问题.展开的坏处是代码量增加,好处是分支预测效率高,占用资源少,同时指令可以充分利用流水线和并行.由于在我的机器上对于这种场景难以复现,我就没提供测试数据对比.但loop-unrolling是分支预测典型的优化方式之一.

最后这个场景代码逻辑是在一个大的int二维数组中按随机顺序造了一些测试的奇偶数序列,然后在循环的条件分支中判断奇偶数,一共有10个case,它们的长度分别是2,2+1,4,4+1,8,8+1,16,16+1,32,32+1,另外测试条件是多表达式.

{% codeblock branch predictor test lang:java %}

  public static void main(String[] args) {
		int x = 0;
		int y = Integer.valueOf(args[0]);
		int[][] randoms = new int[][] {
				new int[]{1,2},
				new int[]{1,2,2},
				new int[]{1,1,2,1},
				new int[]{1,1,2,1,2},
				new int[]{1,1,2,1,1,1,2,1},
				new int[]{1,1,2,1,1,1,2,1,2},
				new int[]{2,1,1,2,1,2,1,2,1,1,1,1,1,2,2,2},
				new int[]{1,1,1,2,2,2,1,1,1,1,2,1,1,2,2,1,1},
				new int[]{1,1,1,2,2,2,1,1,1,1,2,1,1,2,2,1,2,1,1,1,2,2,1,1,2,2,2,2,1,2,1,2},
				new int[]{1,1,1,2,2,2,1,1,1,1,2,1,1,2,2,1,2,1,1,1,2,2,1,1,2,2,2,2,1,2,1,2,1},
		};
		
		System.out.println("branch predictor sequence length:" + randoms[y].length);
		for (int i = 0; i < 1000; i++) {
			for (int j = 0; j < 100000; j++) {
				if (randoms[y][j%randoms[y].length] == 1 || randoms[y][i%randoms[y].length] == 1) {
					x = j+1;
				} else {
					x = j;
				}
			}
		}
		System.out.println(x);
	}
  
{% endcodeblock %}

测试脚本:
{% codeblock  lang:bash %}
  for i in {0..9}
  do
	  echo -e "test case:$i\n"
	  perf stat java GGG $i
  done
{% endcodeblock %}

{% codeblock result %}
test case:0

branch predictor sequence length:2
99999

 Performance counter stats for 'java GGG 0':

        435.919199 task-clock                #    1.028 CPUs utilized          
               135 context-switches          #    0.000 M/sec                  
                37 CPU-migrations            #    0.000 M/sec                  
             4,203 page-faults               #    0.010 M/sec                  
     1,278,164,688 cycles                    #    2.932 GHz                     [51.00%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,649,473,158 instructions              #    1.29  insns per cycle         [76.19%]
       454,492,126 branches                  # 1042.606 M/sec                   [74.86%]
         1,593,299 branch-misses             #    0.35% of all branches         [74.27%]

       0.424036750 seconds time elapsed

test case:1

branch predictor sequence length:3
100000

 Performance counter stats for 'java GGG 1':

        440.982464 task-clock                #    1.017 CPUs utilized          
               134 context-switches          #    0.000 M/sec                  
                 7 CPU-migrations            #    0.000 M/sec                  
             4,202 page-faults               #    0.010 M/sec                  
     1,313,300,847 cycles                    #    2.978 GHz                     [48.14%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,691,221,085 instructions              #    1.29  insns per cycle         [74.32%]
       464,577,389 branches                  # 1053.505 M/sec                   [76.70%]
         1,860,782 branch-misses             #    0.40% of all branches         [75.40%]

       0.433542282 seconds time elapsed

test case:2

branch predictor sequence length:4
100000

 Performance counter stats for 'java GGG 2':

        404.754545 task-clock                #    0.998 CPUs utilized          
               134 context-switches          #    0.000 M/sec                  
                 0 CPU-migrations            #    0.000 M/sec                  
             4,201 page-faults               #    0.010 M/sec                  
     1,205,179,492 cycles                    #    2.978 GHz                     [49.49%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,458,990,354 instructions              #    1.21  insns per cycle         [75.24%]
       376,194,631 branches                  #  929.439 M/sec                   [75.31%]
         1,333,624 branch-misses             #    0.35% of all branches         [75.46%]

       0.405460356 seconds time elapsed

test case:3

branch predictor sequence length:5
99999

 Performance counter stats for 'java GGG 3':

        672.566849 task-clock                #    0.998 CPUs utilized          
               169 context-switches          #    0.000 M/sec                  
                 0 CPU-migrations            #    0.000 M/sec                  
             4,202 page-faults               #    0.006 M/sec                  
     2,008,298,057 cycles                    #    2.986 GHz                     [49.48%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,525,468,338 instructions              #    0.76  insns per cycle         [75.04%]
       407,631,191 branches                  #  606.083 M/sec                   [75.57%]
        23,903,519 branch-misses             #    5.86% of all branches         [75.13%]

       0.673644531 seconds time elapsed

test case:4

branch predictor sequence length:8
100000

 Performance counter stats for 'java GGG 4':

        405.341190 task-clock                #    0.998 CPUs utilized          
               129 context-switches          #    0.000 M/sec                  
                 0 CPU-migrations            #    0.000 M/sec                  
             4,204 page-faults               #    0.010 M/sec                  
     1,207,034,902 cycles                    #    2.978 GHz                     [47.61%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,464,313,449 instructions              #    1.21  insns per cycle         [73.32%]
       373,966,105 branches                  #  922.596 M/sec                   [76.32%]
         1,419,666 branch-misses             #    0.38% of all branches         [76.29%]

       0.406078318 seconds time elapsed

test case:5

branch predictor sequence length:9
100000

 Performance counter stats for 'java GGG 5':

        794.795883 task-clock                #    0.998 CPUs utilized          
               190 context-switches          #    0.000 M/sec                  
                 0 CPU-migrations            #    0.000 M/sec                  
             4,203 page-faults               #    0.005 M/sec                  
     2,373,644,196 cycles                    #    2.986 GHz                     [49.22%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,516,162,160 instructions              #    0.64  insns per cycle         [74.37%]
       402,595,975 branches                  #  506.540 M/sec                   [75.37%]
        36,324,079 branch-misses             #    9.02% of all branches         [75.57%]

       0.796235469 seconds time elapsed

test case:6

branch predictor sequence length:16
99999

 Performance counter stats for 'java GGG 6':

        408.163870 task-clock                #    0.998 CPUs utilized          
               136 context-switches          #    0.000 M/sec                  
                 0 CPU-migrations            #    0.000 M/sec                  
             4,202 page-faults               #    0.010 M/sec                  
     1,214,173,766 cycles                    #    2.975 GHz                     [49.83%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,533,685,373 instructions              #    1.26  insns per cycle         [75.45%]
       412,685,366 branches                  # 1011.078 M/sec                   [75.54%]
         1,366,491 branch-misses             #    0.33% of all branches         [74.87%]

       0.408941737 seconds time elapsed

test case:7

branch predictor sequence length:17
99999

 Performance counter stats for 'java GGG 7':

        837.196938 task-clock                #    0.999 CPUs utilized          
               165 context-switches          #    0.000 M/sec                  
                 8 CPU-migrations            #    0.000 M/sec                  
             4,202 page-faults               #    0.005 M/sec                  
     2,497,386,962 cycles                    #    2.983 GHz                     [49.25%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,514,795,880 instructions              #    0.61  insns per cycle         [75.21%]
       401,526,259 branches                  #  479.608 M/sec                   [75.18%]
        39,563,587 branch-misses             #    9.85% of all branches         [75.67%]

       0.838282370 seconds time elapsed

test case:8

branch predictor sequence length:32
100000

 Performance counter stats for 'java GGG 8':

        747.169398 task-clock                #    1.006 CPUs utilized          
               178 context-switches          #    0.000 M/sec                  
                51 CPU-migrations            #    0.000 M/sec                  
             4,199 page-faults               #    0.006 M/sec                  
     2,231,730,222 cycles                    #    2.987 GHz                     [50.33%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,630,954,802 instructions              #    0.73  insns per cycle         [75.53%]
       438,123,215 branches                  #  586.377 M/sec                   [74.94%]
        30,522,725 branch-misses             #    6.97% of all branches         [74.87%]

       0.742440916 seconds time elapsed

test case:9

branch predictor sequence length:33
100000

 Performance counter stats for 'java GGG 9':

        970.124951 task-clock                #    1.006 CPUs utilized          
               197 context-switches          #    0.000 M/sec                  
                51 CPU-migrations            #    0.000 M/sec                  
             4,202 page-faults               #    0.004 M/sec                  
     2,897,829,158 cycles                    #    2.987 GHz                     [49.89%]
   <not supported> stalled-cycles-frontend 
   <not supported> stalled-cycles-backend  
     1,615,586,420 instructions              #    0.56  insns per cycle         [75.29%]
       432,153,910 branches                  #  445.462 M/sec                   [74.86%]
        51,106,599 branch-misses             #   11.83% of all branches         [75.29%]

       0.964319417 seconds time elapsed

{% endcodeblock %}

以上测试不严谨的地方在于没有多次测试并取平均值,但经过人肉的repeat和肉眼的分析来看,测试数据表现出的特征很明显,所有测试序列长度大于2的结果中,为2的幂次+1的结果都在分支预测上表现不理想,同时当序列长度等于32时,结果要明显高于小于等于16的2的幂次的结果.

这个场景很特别,也有很多值得思考的问题,目前没有很明确的答案,只有一些基本的猜测:

1. 仔细看看,发现if条件中的第二个表达式在内循环中由于i的不变性而保持不变,第一个条件虽然会发生变化,但其值具备很强的可预测性,因为在去掉第二个条件后,miss率降到3%左右.而将第二个表达式前提后,miss率也降低到3%.
   这个现象十分之诡异,也难以解释,准备试试vtune,看能否采集到更精准的数据来分析.
   
2. 计算机喜欢2的幂次
   + 刚毕业那会某大牛对我说,如果你要使用一个数字变量,尽量定义为2的幂次.当然长度16以后的性能剧降能间接的说明了解branch history长度的意义.
   + 2的幂次+1带来的更多的随机性和资源的负担,我只能说记住第一条,但无法量化分析这个现象,如果真要分析,有机会进intel我会好好研究~.

## 结论
1. 2的幂次
2. 优化hotspot,尽量消除不必要的分支,如果必须要使用,注意分析branch内表达式的随机性.
3. 分析分支预测失败因子基于了解当前CPU内部架构的基础上.

## 参考

+ [分支预测器](http://zh.wikipedia.org/zh/%E5%88%86%E6%94%AF%E9%A0%90%E6%B8%AC%E5%99%A8 )  
+ [intel开发者blog-如何避免分支预测失败的开销](http://software.intel.com/en-us/articles/avoiding-the-cost-of-branch-misprediction ) 
+ [某教授对各种类似CPU内部架构的分析和猜测, 专业!](http://www.docin.com/p-86138919.html )
+ [loop unrolling](http://en.wikipedia.org/wiki/Loop_unwinding ) 
+ [分支预测分析](http://www.cs.cmu.edu/afs/cs/academic/class/15740-f02/public/doc/discussions/uniprocessors/branch_pred/msmith_isca95.pdf ) 

intel官方文档我没有找到对于分支预测的详细介绍,很多地方都是一笔带过.这里就不列举了.








