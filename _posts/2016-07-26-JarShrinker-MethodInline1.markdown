---
layout:     post
title:      "JarShrinker MethodInline"
subtitle:   " \"方法内联问题排查\""
date:       2016-07-26 12:00:00
author:     "Jeremy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags: [ASM,Class,Java,Shrink,MethodInline]
---

> “一言不合就朝你扔一段字节码. ”

<p id = "build"></p>
---

### 问题

在做自动方法内联的过程中，发现优化后的包运行时出现NPE问题，经过一系列定位，将问题抽象如下，优化前的代码是


```
package test;
public class Test{
	public static void main(String... args){
		Outer.getInstance().printHelloWorld();
	}
}
```
```
package test;
public class SingleInstance{
	static{
		SingleInstance instance = getInstance(); //或者SingleInstance instance = InstanceHolder.instance;
		instance.printHelloWorld();
	}
	public static SingleInstance getInstance(){
		return InstanceHolder.instance;
	}
	public void printHelloWorld(){
		System.out.println("HelloWorld");
	}
	private static class InstanceHolder{
		private static final SingleInstance instance = new SingleInstance();
	}
}
```
这是一种比较常见的单例写法，比较特殊的是在初始化SingleInstance类时，会在静态初始化块中对单例对象进行一些操作，这里抽象成打印‘HelloWorld’

如果来看生成的字节码，getInstance方法是调用Inner类自动生成access$方法去间接访问instance属性的值，由于jar-shrinker默认会裁剪access$方法和内联指令数较少的方法，所以优化后的代码如下：

```
package test;
public class Test{
	public static void main(String... args){
		SingleInstance.InstanceHolder.instance.printHelloWorld();
	}
}
```
```
package test;
public class SingleInstance{
	static{
		SingleInstance instance = getInstance(); //或者SingleInstance instance = Inner.instance;
		instance.printHelloWorld(); //Crash位置
	}
	public void printHelloWorld(){
		System.out.println("HelloWorld");
	}
	static class InstanceHolder{
		static final SingleInstance instance = new SingleInstance();
	}
}
```
在运行会导致在执行SingleInstance方法的静态初始化块代码时出现instance为null的情况，一开始百思不得其解，后来通过了解类的初始化顺序找到了原因。

在未优化的代码中，Test.main函数会触发jvm去加载SingleInstance类，在执行SingleInstance类静态初始化块时，调用getInstance方法又会触发jvm去加载InstanceHolder类，于是InstanceHolder的静态初始化块被执行，InstanceHolder.instance被赋值，初始化完InstanceHolder类后，getInstance方法返回InstanceHolder.instance就不为空
![Inner-Outer](http://xiaohaolong.com/attach/2016-07-26-JarShrinker-MethodInline1/initializer_1.png "Optional title")
在优化后的代码中，Test.main一开始会触发jvm去加载InstanceHolder类，加载内部类会触发jvm去加载其宿主类，于是SingleInstance类被加载，SingleInstance的静态方法块先于InstanceHolder的静态方法块执行，所以SingleInstance静态初始化块中的getInstance返回null，导致NPE。
![Outer-Inner](http://xiaohaolong.com/attach/2016-07-26-JarShrinker-MethodInline1/initializer_2.png "Optional title")
### 类初始化
为了验证内部类和外部类的初始化顺序，重新写两个简单的小程序：

```
package test;
public class Test{
	public static void main(String... args){
		Outer.getInstance().printHelloWorld();
	}
}
```
```
package test;
public class Outer{
	static{
		Outer instance = getInstance(); //或者Outer instance = Inner.instance;
		System.out.println("Outer static");
	}
	public static Outer getInstance(){
		return Inner.instance;
	}
	public void printHelloWorld(){
		System.out.println("HelloWorld");
	}
	static class Inner{
		static final Outer instance = new Outer();
		static{
			System.out.println("Inner static");
		}
	}
}
```

运行时输出：

Inner static

Outer static

HelloWorld

执行优化后的代码为：

```
package test;
public class Test{
	public static void main(String... args){
		Outer.Inner.instance.printHelloWorld();
	}
}
```
```
package test;
public class Outer{
	static{
		Outer instance = Inner.instance;
		System.out.println("Outer static");
	}
	public void printHelloWorld(){
		System.out.println("HelloWorld");
	}
	private static class Inner{
		private static final Outer instance = new Outer();
		static{
			System.out.println("Inner static");
		}
	}
}
```

运行时输出：

Outer static

Inner static

HelloWorld

类的初始化顺序虽然可以理解成“按需初始化”，但当初始化一个类时，如果其宿主类，接口，基类还未初始化，会先执行这些类型的初始化。这便是由于代码优化导致的类初始化顺序发生变更从而造成的一个比较隐蔽的bug。

### 对业务代码的影响
为了验证这个bug的影响范围，我构造了另一个testcase

```
package test;
public class Test{
	public static void main(String... args){
		System.out.println(Wrapper.getInstance().getI());
	}
}
```
```
package test;
public class Wrapper{
	static{
		Outer instance = getInstance();
		instance.incr();
	}
	public static Holder getInstance(){
		return Holder.instance;
	}	
}
```

```
package test;
public class Holder{
	private int i = 0;
	private static final Holder instance = new Holder();
	private void incr(){
		i++;
	}
	public void getI(){
		return i;
	}
}
```
运行时输出： 1

优化后代码如下：

```
package test;
public class Test{
	public static void main(String... args){
		System.out.println(Holder.instance.getI());
	}
}
```
```
package test;
public class Wrapper{
	static{
		Outer instance = Holder.instance;
		instance.incr();
	}	
}
```

```
package test;
public class Holder{
	private int i = 0;
	private static final Holder instance = new Holder();
	private void incr(){
		i++;
	}
	public void getI(){
		return i;
	}
}
```
运行时输出： 0

如果有人写出这么一串垃圾，那么代码优化后，会由于Wrapper的初始化块没有被执行(Wrapper类根本不需要被加载)，导致业务逻辑都发生异常(数值从1变成0)。

JarShrinker功能最基本要求便是保证业务正确且对于业务透明，所以这种情况必须进行规避，目前的处理逻辑如下：

1 非static方法，可以进行优化

2 static方法，如果字节码中访问了当前类的属性/方法，则可以进行优化

3 static方法，如果字节码中没有访问当前类的属性/方法，但访问的类属性和方法都来自libClass，则可以进行优化

其他情况的static方法都暂时keep


