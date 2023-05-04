---
layout: post
title: "Struts2.3.16升级后无法动态查找action及方法"
# subtitle: "This is the post subtitle."
comments: true
date:  2014-08-22 20:39:47 +0800
categories: 'cn'
---

问题背景及描述：这个问题之前帮别人升级struts2的时候遇到过，也解决了，但是时间过去有点久，以至于一时没想起来，说明随手记笔记很有用啊。。

主要就是如题所示，新版本struts2无法动态查找action，总是报错no action.  

  

解决思路：

1.首先从web.xml中配置的struts的filter入手，进入调试。其他都可以先忽略，重点是mapping，

```java
ActionMapping mapping = prepare.findActionMapping(request, response, true);
```

2.跟踪进入到ActionMapping org.apache.struts2.dispatcher.ng.PrepareOperations，其中有getMapping，从字面上理解就知道是获取mapping的方法了（Java的好处啊），继续跟踪调试；

3.getMapping有多个实现方法，所以要跟着调试一步步走，直接F5进入DefaultActionMapper。方法内部有三个方法，

```java
1.  parseNameAndNamespace(uri, mapping, configManager);
2.  handleSpecialParameters(request, mapping);
3.  return parseActionName(mapping);
```

第一个是Action名和namespace，第二个是参数，第三个才是重点。

4.parseActionName里面有个allowDynamicMethodCalls，默认值是false，如果直接跳过，则mapping中method为null，这就是问题所在。

5.知道问题根源，接下来就是解决，allowDynamicMethodCalls如何赋值为true，还是DefaultActionMapper类中：

```java
1.  @Inject(StrutsConstants.STRUTS_ENABLE_DYNAMIC_METHOD_INVOCATION)
2.  public void setAllowDynamicMethodCalls(String allow) {
3.      allowDynamicMethodCalls = "true".equalsIgnoreCase(allow);
4.  }
```

注入值，struts.enable.DynamicMethodInvocation，所以只需要在struts.xml中加入`<constant name="struts.enable.DynamicMethodInvocation" value="true"/>`；

6.最后，site:struts.apache.org DynamicMethodInvocation，官方文档对这个DMI的介绍主要如下：原本是webwork的特性，用来执行非execute方法的，而他们之所以这样设置为false，主要是两方面考虑，一是安全性，二是和struts1版本的Wildcard特性有冲突。

  

问题解决。