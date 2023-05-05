---
layout: post
title: "java.lang.AbstractMethodError【二】——浅析JVM中invokevirtual与invokeinterface的区别"
# subtitle: "This is the post subtitle."
comments: true
date:   2018-12-11 11:47:43 +0800
categories: 'cn'
---

### 前言
[上一篇文章]({% post_url 2019-03-30-jvm_abstractMethod %})提到了问题的起因，以及简单的介绍了invokeinterface，这篇文章着重于invokevirtual与invokeinterface的区别，同样，由于个人技术水平有限，有些问题暂时没有找到答案，仅在此将其列出。

### 起因
上一篇文章提到了AbstractMethodError的起因是由于DBCP中的DelegatingStatement.isClosed被申明成protected，与Java6及之后的Java版本中的Statement.isClosed有冲突，因此在运行期遇到这样的错误，但是DBCP的版本一直没有调整过，之前一直运行的好好地，怎么突然就报这个错了呢？
于是查阅了当时的git commit log，发现有人将原来的ProxyStatement删掉了，错误便是从这个commit开始的，ProxyStatement是abstract class，其中isClosed被申明成abstract。

### 分析
我们知道Java中interface method会被编译成invokeinterface，而非private的member method（还有super调用）会被编译成invokevirtual，所以看起来问题像是invokeinterface不能解析到DelegatingStatement.isClosed而invokevirtual却能正确解析到DelegatingStatement.isClosed，是不是这两者在JVM实现上有区别？

于是我在上一篇文章的sample code的基础上又增加了个简单的sample code：
```java
public class AbstractTest {

  public static void main(String[] args) {
    System.out.println(new DefaultClassWrapper(new AnotherClass()).getName());
    
    AbstractClass wrapper = new DefaultClassWrapper(new DefaultClass());
    System.out.println(wrapper.getName());
  }

  public static abstract class AbstractClass {
    
    public abstract String getName();
  }

  public static class DefaultClass extends AbstractClass {

    public String getName() {
      return "Default";
    }
  }

  public static class DefaultClassWrapper extends AbstractClass {

    private AbstractClass clazz;

    public DefaultClassWrapper(AbstractClass clazz) {
      this.clazz = clazz;
    }

    public String getName() {
      return clazz.getName();
    }
  }

  public static class AnotherClass extends AbstractClass {

    public String getName() {
      return "Another";
    }
  }
}
```
然后通过JByteMod把两份sample code中的DefaultClass的getName都从public改成了protected，接着使用JVM7运行，结果：
![image](https://user-images.githubusercontent.com/3426457/55272977-f742d080-52ff-11e9-9da7-665845339e59.png)

### 什么原因？
我们知道Java中不允许多重继承，但是允许一个class实现多个interfaces，意味着一个class最多只能有一个父类但可以有多个接口，这也导致了invokevirtual和invokeinterface在实现上的不同，具体来说invokevirtual是通过vtable，而invokeinterface是通过itable。

#### VirtualCall
vtable是JVM实现虚方法调用的核心结构，class的vtable结构继承于super class的vtable，自身的非私有成员方法排在之后，这意味着子类的方法和其抽象父类的同签名方法拥有一样的vtable index，因此在解析invokevirtual符号引用的时候只需要知道方法的vtable index（这里并不要求去从实际类型中获得，只需要从外观类型中获得即可）。
invokevirtual的调用执行可以简单分为三个步骤：
- 从实例对象header中获得_klass
- 从instanceKlass中获得vtable
- 根据index获得对应的vtable entry并执行

举个简单的例子，对于形如：
```java
AbstractClass parent = new ChildClass();
parent.foo();
```
可以分为两个阶段，第一阶段为解析：即找出foo在AbstractClass vtable中的index，第二阶段为执行（动态绑定）：从parent对象header中获得_klass，发现_klass指向的是ChildClass，接着从ChildClass.vtable[index]中获得可执行代码然后执行。
假设AbstractClass.foo是final的呢？对于被final修饰的方法，JLS规定不允许被子类override，因此在解析阶段，JVM有机会对这种方法进行优化，而不用在动态绑定时去查询实际类型的vtable。

虚方法调用的概念大致如此，但在实际情况中，大部分虚方法调用都是单态的，因此除了final之外JVM还使用了[内联缓存（inline caching）](https://en.wikipedia.org/wiki/Inline_caching)进行优化，所谓的内联缓存和方法内联是没有关系的，内联缓存指的是通过记住调用点之前的方法查找结果来加速动态绑定过程，本质上是通过空间换时间。内联缓存根据调用点所缓存的对象类型数量区分为单态内联缓存，多态内联缓存以及超多态内联缓存（复态内联缓存）。
- 单态内联缓存：调用点中缓存的对象类型仅有一种，这是实践中最常见的情况。调用点初始状态为unlinked，此时调用的对象类型会被缓存，当下一次执行动态调用时如果调用对象类型与缓存中的类型比较通过，则执行直接调用目标方法。
- 多态内联缓存：调用点中缓存的对象类型有多种。当调用点为单态时调用对象类型与缓存中的类型比较不通过时，调用点不会回到unlinked状态，而是进入到多态状态，意味着此时调用点会缓存多个对象类型。
- 超多态内联缓存：由于内联缓存需要额外的内存空间且缓存空间大小通常是有限的，因此进入到超多态时，JVM会选择放弃优化，直接查询vtable。（JVM只使用了单态和超多态两种内联缓存，也就是说一旦调用点发现超过一个调用对象类型，则劣化成超多态，从vtable中查找目标方法）

#### InterfaceCall
前面提到Java中仅支持单继承，但允许实现多个接口，意味着对于接口调用，由于无法保证调用类型中目标方法的index与接口类型中目标方法的index一致，因此invokeinterface不能通过vtable来执行方法调用，JVM为此提供另一种类似于vtable的结构叫itable来实现动态绑定。
与invokevirtual通过index查询vtable中的目标方法不同，invokeinterface首先需要保证调用类型实现了目标接口以及接口方法在调用类型中存在，然后遍历调用类型中实现的所有接口直到找到目标接口，一旦目标接口找到，而目标方法在目标接口中itable的index是固定的，只需要再加上目标接口在调用类型中的offset便可以找到可执行代码，进而完成方法调用。
举个简单的例子，对于形如：
```java
InterfaceClass parent = new ChildClass();
parent.foo();
```

解析过的invokeinterface指令会包含InterfaceClass和foo在InterfaceClass itable中的index，执行方法调用时JVM会找到parent对象header中的_klass，发现该instanceKlass指向ChildClass，然后在ChildClass实现的接口列表中找到InterfaceClass，以及InterfaceClass在ChildClass的itable中的offset，接着从ChildClass.itable[offset+index]中找到可执行代码然后执行。
但如果InterfaceClass中定义了Object类中例如hashCode方法时，itable并不会为这种方法分配空间，因为这其实已经是invokevirtual调用了，因为vtable中一定存在hashCode的entry，并且index与Object的index一致。

同样，JVM对invokeinterface也有内联缓存优化，且优化方式与invokevirtual几乎是一样的（[InterfaceCalls](https://wiki.openjdk.java.net/display/HotSpot/InterfaceCalls)中有提到对查找itable的优化），只有在劣化至超多态的情况下，由于itable的查找效率略低于vtable，invokeinterface的性能会差于invokevirtual。

### 总结
《深入理解JVM》中曾经提到invokevirtual会在找到目标方法时检查access scope，不通过的时候会抛出IllegalAccessError，但实际查阅JVMS7时，invokevirtual完全没有提到会抛IllegalAccessError，倒是JVMS8有提到，不允许使用父类作为外观类型来调用父类中的申明为protected的方法，原文为：
> if the resolved method is a protected method of a superclass of the current class, declared in a different runtime package, and the class of objectref is not the current class or a subclass of the current class, then invokevirtual throws an IllegalAccessError.

那么AbstractClass可以正常运行便得到了合理的解释，因为protected并不会阻止invokevirtual执行方法调用，至于InterfaceClass为什么会抛出AbstractMethodError，跟上文最后的问题一样未知...

### 参考
1. [https://en.wikipedia.org/wiki/Virtual_method_table](https://en.wikipedia.org/wiki/Virtual_method_table)
2. [https://wiki.openjdk.java.net/display/HotSpot/VirtualCalls](https://wiki.openjdk.java.net/display/HotSpot/VirtualCalls)
3. [https://wiki.openjdk.java.net/display/HotSpot/InterfaceCalls](https://wiki.openjdk.java.net/display/HotSpot/InterfaceCalls)
4. [https://www.zhihu.com/question/34846173/answer/60302017](https://www.zhihu.com/question/34846173/answer/60302017)
5. [http://staff.ustc.edu.cn/~bjhua/courses/compiler/2014/readings/interface.pdf](http://staff.ustc.edu.cn/~bjhua/courses/compiler/2014/readings/interface.pdf)