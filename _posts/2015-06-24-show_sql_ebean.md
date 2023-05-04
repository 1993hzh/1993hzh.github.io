---
layout: post
title: "Play! Framework—show sql with Ebean in play2.4"
# subtitle: "This is the post subtitle."
comments: true
date:  2015-06-24 16:42:04 +0800
categories: 'cn'
---

Ebean是play内置的ORM框架，play2.0-2.3版本默认用的数据库连接池是bonecp，因此ebean的sql log可以通过配置application.conf，设置

```
1.  db.default.logStatements=true
2.  logger.com.jolbox=DEBUG
```

但是在2.4版本开始，play团队不再建议通过application.conf中配置logger，推荐使用conf/logback.xml进行配置，且由于2.4版本开始play默认使用hikaricp作为默认的数据库连接池， com.jolbox不再起效，另外hikaricp不提供log sql的方法，所以将com.jolbox更改为com.zaxxers是无效的（坑太多了不忍直视）。说了这么多，其实解决办法很简单，ebean中是通过TransactionManager中的SQL_LOGGER进行sql log的， 所以只需要在logback.xml中设置

`<logger name="org.avaje.ebean.SQL" level="TRACE"/> # level可以根据需要更改`

另外stackoverflow中还有介绍一些其他办法的， 一并回答一下，例如

1. `Ebean.getServer(null).getAdminLogging().setLogLevel(LogLevel.SQL);` 原因：getAdminLogging在ebean4.6版本中没有该api；

2. 使用jdbcdslog的， 即使在logback.xml中设置了logger也无效，具体原因不明。
