---
layout: post
title: "String intern你必须知道的几点"
subtitle: 'java String intern方法解析'
author: "Starboyate"
header-img: "img/singleton.jpg"
multilingual: true
tags:
  - Java
  - 基础
---

## 一、前言
> String类型是java里面一个很特殊的存在，里面涉及到了很多使用细节以及很多基础知识，
这里主要分析下String里的intern方法。OK,let's go!

<br/>

## 二、正文
##### 1.字符串常量池
> 字符串常量池其实就是一个固定长度的HashTable,是java提供的系统级别的一个cache，为了节省内存以及提升性能

<br/>

下面来看一下一个例子,相信在笔试或者面试的时候经常会遇到这么一个问题，String a = new String("1")这行代码到底创建了多少个对象？
- 第一种情况，如果当前字符串常量池不存在"1"，那么就会创建两个对象，一个在常量池中，一个在堆中。
- 第二种情况，如果当前字符串常量池已经存在了"1",那么只会创建一个对象，这个对象在堆中。

<br/>



通过上面的例子可以看出，其实每次使用魔法值的时候，都会往常量池中存储一份，这样以后使用的时候可以直接从常量池中使用，
并且常量池的一个加载都是在编译器就已经确认好的，性能提升不少。

<br/>

##### 2.intern() 含义
> 官方注释:
When the intern method is invoked, if the pool already contains a string equal to this String object as determined by the equals(Object) method, 
then the string from the pool is returned. Otherwise, this String object is added to the pool and a reference to this String object is returned.
<br/>
上面的意思大概就是，使用intern方法的时候，会去常量池中判断是否存在这个字符串对象，如果不存在，就往常量池中存储一份，然后返回池中对应的对象。
如果存在了，那么直接返回池中存在的字符串对象。

<br/>

##### 3.intern() 实战
场景一:
```java
public class Main {
	public static void main(String[] args) {
		String a = new String("1");
		a.intern();
		String b = "1";
		System.out.println(a == b);
	}
}
```

<br/>

来具体分析上面的代码:
- String a = new String("1");这一行，创建了两个对象，一个在常量池中，一个在堆中，a指向了堆中的String对象。

<br/>

- a.intern();这一行，是把a这个字符串对象放入到常量池中，这里注意，前面已经在常量池进行了存储，所以这里是不会再把a放入到常量池中。

<br/>

- String b = "1";这个b是指向了常量池中

<br/>

- System.out.println(a == b);通过上面的分析，相信很容易看出是false，因为a和b不是同一个对象，一个指向的是堆中的，一个指向的是常量池中。

<br/>

场景二:
```java
public class Main {
	public static void main(String[] args) {
		String a = new String("1").intern();
		String b = "1";
		System.out.println(a == b);
	}
}
```

<br/>

同样的，来具体分析上面的代码:
- String a = new String("1").intern();首先在这里，还是创建了两个对象，然后调用了intern()方法，在前面介绍intern的定义的时候，当调用
intern方法的时候，会返回在常量池中的对象，所以这时候a其实指向的是常量池中的。

<br/>

- String b = "1";这没什么好说的，也是指向常量池中

<br/>

- System.out.println(a == b);那么两个都是指向同一个地方，肯定为true

<br/>

> 这里需要注意一点，我们就是当任意一个字符串，a.equals(b)的时候，a.intern() == b.intern() 为true。

<br/>

场景三:
```java
public class Main {
	public static void main(String[] args) {
		String a = new String("1") + new String("2");
		a.intern();
		String b = "12";
		System.out.println(a == b);
	}
}
```

<br/>

这里可能有的人会输出false，有的人输出true，这是为什么呢？
其实这是因为不同版本(具体是jdk7之前和jdk6以后)的java内存划分有所区别，导致了输出的内容也有所差异，下面让我们来分析下上面的代码
- String a = new String("1") + new String("2");这里其实创建了5个对象，常量池中的"1"和"2"对象以及堆中的"1","2"以及"12"对象,
这里为什么常量池中没有"12"这个字符串对象呢，其实是因为当我们使用变量拼接的时候，java会帮我们优化成StringBuilder对象进行append,
最后在调用toString返回一个对象给我们，所以常量池中是没有这个字符串对象的。下面，我就以String a = new String("1") + new String("2")进行反编译
![intern反编译图片](/img/intern.png)

<br/>

- a.intern();这里会把a字符串对象存储到常量池中(注意，这时候常量池中是没有"12"这个字符串对象的)。在jdk7之前，常量池是在perm space中的，也就是常说的永久代或者方法区中，
当我们调用intern()的时候，其实是重新把a的值copy到常量池中，这时候，因为a指向的是堆中的引用，而b是常量池中那份copy过来的值，
所以肯定是不相等的。在jdk6以后，常量池是移出了perm space放入到堆中了，并且调用intern方法的时候是直接存储了堆中的引用，所以b和a比较是true。

<br/>

- 相信看完上面这两点的分析，可以很清晰的知道自己当前输出是true还是false了。

<br/>

场景四:
```java
public class Main {
	public static void main(String[] args) {
		String a = new String("1") + new String("2");
		String b = "12";
		a.intern();
		System.out.println(a == b);
	}
}
```

<br/>

我们来分析最后的这个例子:
- String a = new String("1") + new String("2");这里跟之前一样就不过多分析了

<br/>

- String b = "12";这里其实会在常量池中存储一个字符串对象

<br/>

- a.intern();调用intern()方法

<br/>

- 最后输出是false，相信看了之前那么多分析，也明白清楚了，因为a指向的是堆中的，而b是在常量池中的，虽然调用a.intern方法，
但是由于常量池中已经存在了，所以不会再去存储一份，自然而然输出也是false。

<br/>

##### 4.关于intern的使用
> 看了上面的分析，我相信已经对于intern这个方法的作用了解的很清楚了，下面来说说为什么要提供这么一个方法让我们可以把
字符串放入到常量池中呢？前面也说了其实常量池相当于缓存的作用，那缓存肯定是可以提升我们的一个访问速度，所以使用intern可以
提升我们访问的速度。但也存在相应的坑，下面具体道来。

使用intern注意事项：
```text
   jdk7之前String常量池的长度为1009，当常量池满了以后，就无法往常量池存储。所以很多一些开源项目为了解决这个问题，都会
自己手动维护一个StringCache,具体的可以看eureka里的StringCache通过WeakHashMap来做StringPool。但在jdk6之后，64位系统的
StringTableSize为60013，32位的还是1009，当然如果还不满足，可以通过-XX:StringTableSize参数来进行修改变更
```


## 三、尾语
>相信通过上面的一系列的分析，也已经彻底掌握了intern的使用，也知道在使用字符串的时候，为了实现复用和速度可以通过
intern方法达到这样的目的，但是要明白但我们使用的是一个魔法值的时候也就是常量的时候，其实会自动在常量池中创建存储，
不需要我们去手动调用intern方法。万丈高楼平地起，基础牢固，方能进阶！





