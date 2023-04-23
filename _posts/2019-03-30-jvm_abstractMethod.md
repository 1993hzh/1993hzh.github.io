---
layout: post
title: "java.lang.AbstractMethodError【一】——浅析JVM7和JVM8关于invokeinterface的区别"
# subtitle: "This is the post subtitle."
date:   2019-03-30 18:04:54 +0800
background: '/assets/images/bg-index.jpg'
categories: 'Chinese'
---

### 前言
这篇文章很早就打算写了，但是因为之前没有保存导致写了一半的文章丢失，后来在内部做技术分享的时候才想起来这件事，于是时隔几个月之后又重新整理了一下，分成了两篇文章，这篇文章主要想聊一下invokeinterface在JVM7和JVM8中的小区别，[下一篇]({% post_url 2018-12-11-jvm_abstractMethod %})重点关注invokeinterface和invokevirtual的区别，由于个人技术水平有限，有些地方也不甚了解，只能将问题抛出，希望以后能有机会找到答案。

### 起因
几个月前，我还在SF的时候，同事在Jenkins上跑CI的时候遇到java.lang.AbstractMethodError，看了一眼callstack，发现错误出在DBCP的DelegatingStatement.isClosed方法上，在本地打开代码看不出问题，于是google了一把，有人提到可能是由于jar包版本导致的兼容性问题，具体一点来讲，Statement.isClosed方法是在JDBC4中新增的方法（除了Statement.isClosed，JDBC4还增加了很多其他方法），而DBCP1.2.2早于JDBC4，尽管其DelegatingStatement提供了一个protected的isClosed方法，但是在Java6及之后的JVM中，调用形如：

> Statement stmt = new DelegatingStatement(...);
> stmt.isClosed();

都会由于JVM解析不到DelegatingStatement.isClosed方法而抛出AbstractMethodError。

### 分析
JLS要求子类override方法时scope不能小于父类中方法的scope，违反这一规则会在编译时就报错，但是JVMS是如何处理这种情况的呢？换句话说，如果子类有一方法，其方法名和方法描述符与父类一致，但修饰符不一样，JVM会认为这是合法的overriding吗？进一步说，JVM是如何解析class中方法的符号引用的呢？

于是我写了个简单的test code
```java
public class InterfaceTest {

  public static void main(String[] args) {
    System.out.println(new DefaultClassWrapper(new AnotherClass()).getName());

    InterfaceClass wrapper = new DefaultClassWrapper(new DefaultClass());
    System.out.println(wrapper.getName());
  }

  public static interface InterfaceClass {

    String getName();
  }

  public static class DefaultClass implements InterfaceClass {

    public String getName() {
      return "Default";
    }
  }

  public static class DefaultClassWrapper implements InterfaceClass {

    private InterfaceClass clazz;

    public DefaultClassWrapper(InterfaceClass clazz) {
      this.clazz = clazz;
    }

    public String getName() {
      return clazz.getName();
    }
  }

  public static class AnotherClass implements InterfaceClass {

    public String getName() {
      return "Another";
    }
  }
}
```

然后通过JByteMod把DefaultClass的getName从public改成了protected，接着使用两个不同版本的JVM运行

JDK7中运行结果：
![image](https://user-images.githubusercontent.com/3426457/55274899-d20e8c00-5318-11e9-97d5-98bb7a2803fb.png)

在JDK8中运行结果：
![image](https://user-images.githubusercontent.com/3426457/55274898-cfac3200-5318-11e9-9a2e-e8f3c9f9ba73.png)

### 什么原因？
一样的class文件却在两个不同版本的JVM中抛出了不同的错误，有些令人费解，于是查阅了JVM规范，发现一些不一样的地方：
#### JVM7中关于invokeinterface如何解析符号引用的说明：
- 如果class（实际类型）中包含了名称和描述符与待解析的方法一致的成员方法，那么就停止查找过程
- 如果上一步没有找到，但class有父类，那么从下往上递归的查找父类中的成员方法，直到找到对应的成员方法或没有父类
- 如果递归查找未找到对应的成员方法，那么抛出AbstractMethodError

##### 符号引用的解析时可能出现的runtime exception：
- 如果class（实际类型）没有实现该interface那么抛出IncompatibleClassChangeError
- 如果没有找到合适的成员方法，那么抛出AbstractMethodError
- 如果选中的成员方法修饰符不是public，那么抛出IllegalAccessError
- 如果选中的成员方法是abstract，那么抛出AbstractMethodError

#### JVM8中关于invokeinterface如何解析符号引用的说明：
- 如果class（实际类型）中包含了名称和描述符与待解析的方法一致的成员方法，那么就停止查找过程
- 如果上一步没有找到，但class有父类，那么从下往上递归的查找父类中的成员方法，直到找到对应的成员方法或没有父类
- **如果class中所有父级interfaces中只有一个非abstract的maximally-specific method与待解析的方法名称和描述符一致，那么查找成功 [[1]](https://jvilk.com/blog/java-8-specification-bug/)**
- 如果以上步骤都没有解析到合适的成员方法，那么抛出AbstractMethodError

##### 符号引用的解析时可能出现的runtime exception：
- 如果class（实际类型）没有实现该interface那么抛出IncompatibleClassChangeError
- 如果查找步骤1和2中所选中的成员方法修饰符不是public，那么抛出IllegalAccessError
- 如果查找步骤1和2中所选中的成员方法是abstract，那么抛出AbstractMethodError
- 如果查找步骤3找到多个非abstract的maximally-specific method，那么抛出IncompatibleClassChangeError
- 如果查找步骤3没有找到非abstract的maximally-specific method，那么抛出AbstractMethodError

JVM8的方法解析步骤3是在JVM7中不存在的，要知道增加这一个步骤的原因，首先得知道什么是maximally-specific method，根据JVM8规范中的定义：
- 必须是接口中申明的方法
- 不能被private或static修饰
- 该方法不能在子接口中重复出现

因此，这里所指的非abstract的maximally-specific method其实就是说Java8中新增的default method，所以如果在父级interfaces中找到多个default methods，就会抛出IncompatibleClassChangeError，这跟JLS描述的是一致的。

JDK8中的IllegalAccessError在上面可以得到解释，因为DefaultClass.getName的byte code被我修改成了protected，所以当解析到DefaultClass.getName时发现修饰符不是public就抛了这个错。
然而，JDK7中也有相同的规则，却为什么抛了AbstractMethodError？难道是因为没有在DefaultClass中找到匹配的成员方法？可是原文明明描述的是`if no method matching the resolved name
and descriptor is selected, invokeinterface throws an
AbstractMethodError`，这里的方法名和方法描述符应该是匹配的。
// 这个问题我暂时还没有找到答案，待续