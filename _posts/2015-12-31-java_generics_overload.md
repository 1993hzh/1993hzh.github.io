---
layout: post
title: "Java泛型遇到重载"
# subtitle: "This is the post subtitle."
comments: true
date:  2015-12-31 06:25:55 +0800
categories: 'cn'
---

今天看到《Scala in Depth》中默认参数值对重载的影响，突然想起Java中有个很tricky的重载，第一次看到是在《深入理解Java虚拟机》中，叫Java泛型遇到重载，作者还特地举了个栗子

```java
1.  import java.util.List;
2.
3.  public class Test {
4.    public int method(List<Integer> list) {
5.      return 0;
6.    }
7.
8.    public String method(List<String> list) {
9.      return "";
10.   }
11. }
```

当然，作者也不忘细心提醒，这种trick只能在sun jdk6下才可以编译通过，并且建议不要写这种毫无美感丧心病狂反人类反社会的代码。

然后呢，我今天在逛知乎的时候又看到这个题目，于是花了点时间去研究了下，结果是，oracle jdk6可以正常编译运行，7、8则会编译报错。

首先，按道理这个本来就应该报错，从Java语言层面来说，方法重载依赖于相同的方法名、不同的参数个数、类型、顺序，而`List<Integer>`和`List<String>`类型擦除后都为`List<E>`，从而不符合方法重载的要求。

但是，为什么会说这种依赖返回值可以通过甚至正常运行，原因在于，编译后的俩个方法在class中的signature分别为

> (Ljava/util/List<Ljava/lang/Integer;>;)I
> 
> (Ljava/util/List<Ljava/lang/String;>;)Ljava/lang/String;

它们可以合法的共存在一个class文件中。

从jdk7开始呢，编译期做了check，保证了behavior一致，所以就报错了

参考链接：[http://bugs.java.com/view_bug.do?bug_id=6182950](http://bugs.java.com/view_bug.do?bug_id=6182950)