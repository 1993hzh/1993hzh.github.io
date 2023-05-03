---
layout: post
title: "几种分页方式在不同使用场景下的优劣势"
# subtitle: "This is the post subtitle."
comments: true
date:   2018-10-14 13:59:35 +0800
background: '/assets/images/bg-index.jpg'
categories: 'cn'
---

分页的主要使用场景大致分为两种，一种是topN查询，这种在实时响应系统中比较常见，另一种是全量查询，在批处理系统中比较常见。

全量查询的分页实现又大致分为两种，一种是基于DB提供的分页API，另一种是从application层提供分页功能，基于application层做分页有多种做法，例如缓存全部result批量读、缓存key做二次查询、基于有序的unique key做条件限制。

### DB提供的分页
因为在项目中常用的是Oracle，这里便用Oracle的ROWNUM举例，不同数据库实现略有不同，但总体上分页的思路是类似的。
```sql
select * from (
    select a.*, ROWNUM r from (%QUERY%) a where ROWNUM <= offset+limit
) where r >= offset;
```
```ROWNUM <= offset+limit```在执行计划中会转化成COUNT STOPKEY的操作，意味着总数为offset+limit条数据会被读取，而最外层的SQL中```ROWNUM>=offset```会截取已读取的数据量，简单来说就像我们要读取一个单链表中的部分元素，我们必须要从head开始读取并用ROWNUM这种伪列来保存当前元素的index。这种分页API有很明显的缺陷，那就是top pages会快于bottom pages，毕竟bottom pages总是要比top pages读更多的数据，即使都是逻辑读，如果bottom pages会引发物理读，那就会更慢。但这也不是绝对的，在Oracle中有两种查询优化模式，一种叫first_rows(n)，还有一种叫all_rows，在Oracle11中默认的是all_rows，first_rows(n)会尽可能快的返回前n条数据，而all_rows则不做这种优化，因此在某些情况下，比如SQL本身足够复杂查询时间达到分钟级甚至更高，那么在all_rows模式下，top pages的性能相比bottom pages就不会有明显的不同。

在Oracle中事务隔离级别默认是READ_COMMITTED，为了保证可重复读，引入了rollback segment，这其实是undo log的一种，undo log file的大小是有限的，如果SQL执行时间过长容易出现undo log被覆写，ORA-01555就是这种情况下出现的错误。
把每页的查询提出来放到单独的事务中做是一种很好的思路，但是由于read consistency无法保证，会出现record missing，record duplication。

### Two-Step分页
简而言之，第一步先缓存结果集的key，第二步再从缓存的key中做分页查询，由于查询结果集被限制在limit个keys中，从而解决了不同optimizer_mode下ROWNUM的稳定性问题，与此同时由于第一步在查询开始时做一个快照，因此如果有并发的事务对同一结果集做insert或delelte也不会对此次查询造成影响，从而保证了read consistency。
这种查询方式由于需要缓存key，因此需要结合实际业务需求考量，假设我们缓存的key是ROW_ID，long类型所占内存为8bytes，因此1M个key放到collection所占用的内存不到32MB（关于JVM中对象的内存占用计算请看#2），当然如果数据量更大一些或者缓存的key类型所占内存大的时候，我们可以考虑引入额外的缓存服务，比如使用临时表或者redis。

### 基于有序UK的分页
这种分页方式可以说是DB层分页与Application层分页的结合，举个简单的SQL例子。

> 采用ROWNUM分页的SQL：
```sql
select * from (select a.*, ROWNUM r from (select * from %table% where %condition% order by %key% asc) a where ROWNUM <= offset+limit) where r >= offset;
```

> 采用有序UK的分页SQL：
```sql
select a.* from (select * from %table% where %condition% and %key% > '&last_key_in_previous_page' order by %key% asc) a where ROWNUM <= limit;
```

这种分页方式在应用层只需要缓存一个UK，因此对server内存的影响远小于Two-Step分页，但其局限性在于需要有一个unique key，因此如果UK对应的record被并发物理删除便会出现问题，而且这种查询方式也必须由DB事务保证一致性读。


以上几种分页方式便是我在项目开发过程中结合实际需求整理出来的对比，由于个人水平有限，如有不足或错误之处，还望可以指出。