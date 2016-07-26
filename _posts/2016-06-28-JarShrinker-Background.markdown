---
layout:     post
title:      "JarShrinker Background"
subtitle:   " \"一个代码优化工具的背景\""
date:       2016-06-28 12:00:00
author:     "Jeremy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags: [ASM,Class,Java,Shrink]
---

> “一言不合就朝你扔一段字节码. ”

<p id = "build"></p>
---
## 背景

当前对于体量较大的安卓应用都存在一个老大难问题，方法数超过65536，除了拆分成多dex这种方案外，还可以借住一些工具来裁剪方法，比如proguard和redex，下面会具体说说这两个工具存在的问题。

首先来说优化场景，目前有几类方法是明确可以优化：

#### access$方法

内部类方法直接访问外部类或者外部类父类的不可见方法/属性，比如下面的例子

```
	public class Outer{
		private int fieldA;
		public class Inner{
			public int getFieldA(){
				return fieldA
			}
		}
	}
```
在编码时Java语法允许内部类去访问外部类的任一方法/字段，但在经过编译后，生成的是Outer和Outer$Inner是两个独立的类，jvm不允许Outer$Inner的getFieldA方法直接去访问Outer类的私有属性fieldA, 所以javac在编译时会自动生成一个access$XXX方法，用这个方法来间接实现对私有属性/方法的访问，还是上面的栗子，下面截图的标注已经解释得足够清楚，对于客户端来说，这些access$方法完全可以通过修改外部类方法/字段的访问属性来优化，上面的例子如果将fieldA的modifier修改为default或更高，则Outer$Inner类会被javac认为与Outer类处于同一个package从而可以直接访问
![Alt text](http://xiaohaolong.com/attach/2016-06-28-JarShrinker/outer.png "Optional title")
![Alt text](http://xiaohaolong.com/attach/2016-06-28-JarShrinker/outer$inner.png "Optional title")

#### getter/setter方法

在Android编程上，如果getter/setter方法没有加上同步等限制，是应该直接将属性访问描述符改为public，并将getter/setter方法去掉，但是很多开发人员仍然会用写Java POJO类的方式来自动生成getter/setter方法，这为客户端贡献了茫茫的方法数。

#### 方法内联
在编程过程中，出于设计的需求或者单一功能的原则，会将很多代码块根据业务逻辑拆分成多个方法，在设计上是无可厚非的，但是在客户端上的确带来不少的方法数，这些指令数较少或者只有一处调用的函数可以通过自动内联来优化，都是可以裁剪的猎物

#### 统计
然后，问题来了，以上三类方法到底贡献了多少方法数，按照我所服务的客户端的统计，这三类方法大概占主dex总方法数的8%左右，同时在运行时，这些方法调用也会带来一定的性能开销。

## 优化方案

#### 人肉修改

发封邮件，让所有开发人员将外部类会被内部类访问的方法/属性访问修饰符修改为default(父类的对应修改为public)，来避免javac生成access$方法； 优化所有pojo类，去掉所有getter/setter。但是这会给很多开发
人员带来一种不安全感，比如我 = =！我这个方法就应该是private，你凭什么一定要我修改成public，让我的接口暴露出去，让我的属性裸奔。而且这种方法对于一个体量很大的app，也会造成很大的人肉成本。

#### proguard/redex 

这两个工具其实并没有针对这三类方法做专门的优化，而是统一用方法内联的方式来优化。方法内联都是针对指令数比较少或者只有一处调用的地方，照理来说是比较通用，但是这两个工具都存在自身的问题，一个一个道来 

1）proguard：在做MethodInline时，proguard并不能处理访问权限下降的情况，依旧以上面的Outer & Outer$Inner 为例子，proguard在判断access$000方法是否可以内联时，发现方法内部访问了一个private属性，这时它就懵逼了，直接跳过，判断方法贴在下面。至于为啥不直接修改proguard来增强它的功能，相信看过proguard源码的同学都会望而却步。。。
![Alt text](http://xiaohaolong.com/attach/2016-06-28-JarShrinker/proguard.png "Optional title")

2）redex:  redex针对函数内联能够处理上面所描述proguard不能处理的情况，虽然是比较简单粗暴地将所有被访问字段/方法修改为public，但是好歹解决了问题 (这估计应该也是facebook blog解释redex在methodInline方面与proguard存在的区别之处)。 然而redex存在两个问题：1 在扩展方法寄存器上策略太过简单粗暴，会导致很多可以优化的地方被跳过，2 更大的问题在于处理是针对dex文件，目前遇到的现状是dex都打不出来，好囧= =! 虽然可以通过  multidex -> redex -> dexMerge来解决，但成本就变高了，非常不直接。而且目前redex这个工具貌似并不成熟，外部很多小app在经过redex处理后都跑不起来，而且使用C++编写后续维护成本估计也比较高。
![Alt text](http://xiaohaolong.com/attach/2016-06-28-JarShrinker/redex.png "Optional title")

#### JarShrinker
于是在打不出包的压力下决定自己来实现 = =！目前在实验的JarShrinker工具，可以针对这三个方面对jar包进行优化，介绍如下：
![Alt text](http://xiaohaolong.com/attach/2016-06-28-JarShrinker/jarShrinkerDesign.png "Optional title")

简单来说就是输入的是一堆原始的jar包，输出是一堆优化后的jar包。

具体一些是通过指令序列来确定可擦除的方法(目前就是access$,getter,setter)，同时结合多态/不同jdk编译产物等情况的判断，对所有调用方的对应调用指令进行修改，同时提升被内联方法的访问属性。 

更具体的麻烦私下跟我交流 ^_^
      

目前的实验结果是主dex能够减少5500+的方法数，已经完成技术灰度，表现比较正常。这个工具可以与proguard配合，同时结合打包流程，实现安卓客户端的方法精简和性能优化。


