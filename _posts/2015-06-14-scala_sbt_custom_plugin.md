---
layout: post
title: "Play! Framework——idea Scala Plugin 1.5.2(custom sbt-launcher设置无效)"
# subtitle: "This is the post subtitle."
comments: true
date:  2015-06-14 21:47:26 +0800
categories: 'cn'
---

sbt或者activator默认在windows用户目录下创建.sbt和.ivy2文件夹，里面存放jar包等， 这里想介绍的是如何自定义lib目录。

  

首先，命令行可以通过创建`%SBT_OPTS%`环境变量指定

```sh
1.  -Dsbt.global.base=E:/activator/.sbt
2.  -Dsbt.ivy.home=E:/activator/.ivy2
3.  -Dsbt.boot.directory=E:/activator/.sbt/boot
```

idea中的 Scala Plugin1.5.2支持play2项目可以定义自己的custom sbt-launcher.jar，但是创建完项目之后进行compile的时候custom sbt-launcher.jar根本不起效，无从知道原因，猜测是这个插件的bug， 可以试试修改default sbt-launcher.jar，路径一般为

```sh
1.  %USER_HOME%/.IntelliJIdea14/config/plugins/Scala/launcher/sbt-launch.jar/sbt/sbt.boot.properties
2.  [boot]directory: E:/activator/.sbt/boot
3.  [ivy]ivy-home: E:/activator/.ivy2
```

这样编译就没有问题了，希望插件的bug能赶紧修复好吧。（可以参照我在segmentfault中的回答 ：[http://segmentfault.com/q/1010000002899390/a-1020000002915171](http://segmentfault.com/q/1010000002899390/a-1020000002915171)）
