---
layout: post
title: "浅析Spring injection支持的几种annotation的实现"
# subtitle: "This is the post subtitle."
date:   2018-10-15 16:44:03 +0800
background: '/assets/images/bg-index.jpg'
categories: 'cn'
---

## 前言
Spring中支持三种注入方式，分别是构造器注入，属性注入，方法注入，这三者之间的优缺点本文不做阐述，有兴趣的可以参考这两篇文章，我觉得写的不错，[Setter injection vs constructor injection ](https://spring.io/blog/2007/07/11/setter-injection-versus-constructor-injection-and-the-use-of-required/)，[浅谈spring为什么推荐使用构造器注入](https://cloud.tencent.com/developer/article/1150501)。
其中，构造器注入发生在createBeanInstance期间，而属性注入和方法注入则发生在populateBean期间，整个getBean的时序图大致如下：![](http://on-img.com/chart_image/5bc4534ce4b015327b07ca7c.png)

## Java standard annotation
### ```@Inject```
最常见也是最常用的注解，[JSR-330](https://www.jcp.org/en/jsr/detail?id=330)中引入，Spring在AutowiredAnnotationBeanPostProcessor中实现了对```@Inject```的支持，AutowiredAnnotationBeanPostProcessor是InstantiationAwareBeanPostProcessor的一种实现。
其中属性注入是通过AutowiredFieldElement，方法注入是通过AutowiredMethodElement，实现方式实际上与```@Autowired```一样，顺带说一句```@Value```也是在这里实现的。
![](http://on-img.com/chart_image/5bc57a48e4b0bd4db96559b9.png)

### ```@Resource```
[JSR-250](https://jcp.org/en/jsr/detail?id=250)中引入的注解，Spring在CommonAnnotationBeanPostProcessor中实现了对```@Resource```的支持。
```@Resource```的实现方式与```@Inject```略有不同，并不是通过重写InjectedElement.inject而是重写了InjectedElement.getResourceToInject。
![](http://on-img.com/chart_image/5bc57b4ce4b08faf8c7d9122.png)

## Non Java standard annotation
### ```@Autowired```
```@Autowired```是Spring自己提供的annotation，其功能几乎与```@Inject```相同，实现也一样，在上文中已经解释过，顺带说一句，能用```@Inject```就不要用```@Autowired```，毕竟```@Autowired```是与Spring绑定在一起的。

## 总结
- @Autowired通过类型进行注入，如果该类型有多种实现，则需要通过@Qualifier指定bean name，如果没有找到@Qualifier，则默认使用field name
- @Inject与@Autowired几乎没什么使用上的区别，除了@Autowired允许null（即required=false）
- @Resource与前面两种不一样的是优先使用name注入，如果找不到对应的bean则尝试通过type注入，最后才会使用@Qualifier（如果有的话）