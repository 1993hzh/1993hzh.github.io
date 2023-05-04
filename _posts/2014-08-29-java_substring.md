---
layout: post
title: "JDK6与JDK7中substring方法的性能问题"
# subtitle: "This is the post subtitle."
comments: true
date:  2014-08-29 11:37:08 +0800
categories: 'cn'
---

JDK6：

```java
1.  public String substring(int beginIndex, int endIndex) {
2.      //check boundary
3.      return new String(offset + beginIndex, endIndex - beginIndex, value);
4.  }

6.  String(int offset, int count, char value[]) {
7.      this.value = value;
8.      this.offset = offset;
9.      this.count = count;
10.  }
```

JDK7：

```java
1.  public String substring(int beginIndex, int endIndex) {
2.      //check boundary
3.      int subLen = endIndex - beginIndex;
4.      return new String(value, beginIndex, subLen);
5.  }

7.  public String(char value[], int offset, int count) {
8.      //check boundary
9.      this.value = Arrays.copyOfRange(value, offset, offset + count);
10.  }
```
  

差别就在于:

JDK6中子串指向的还是父串，只是位置有所不同，这在父串很长同时父串并没有再被使用的情况下，是对内存的消耗。

JDK7中则是copy生成了一个新的array来指向子串，copyOfRange主要是调用了System中的arraycopy方法，这个方法是native的。