---
layout: post
title: "由OGNL的隐式类型转换引发的一个bug"
# subtitle: "This is the post subtitle."
comments: true
date:   2019-05-28 20:28:14 +0800
background: '/assets/images/bg-index.jpg'
categories: 'cn'
---

### 前言
弱类型语言的隐式类型转换是其立足之本，也是让程序员最头疼的特性，如果对其内部的类型转换规则不熟悉的情况下，往往导致线上出现始料不及的bug，比如众所周知的JavaScript中的“==”和“===”，最近在写Mybatis的dynamic SQL也遇到了这样的问题。

### 起因
偶然收到一个用户提的一个问题，试图将一个number类型的字段值改为0没有成功，代码逻辑非常简单，将前端传来的request反序列化成内存对象，然后转成持久层对象，通过DAO进行持久化，没有任何复杂的业务逻辑，并且改成其他值都可以正确保存，唯独0不可以。

### 分析
一开始怀疑是前端传来的JSON对象中的字段反序列化出了问题，将字符串0序列化成了空值，然而实际调试时发现不是这个问题，内存对象中的number字段值是正确的反序列化成了0，接着猜测是Mybatis的dynamic SQL引发的这个问题，找到了表达式片段：```<if test="menu.version != null and menu.version != ''">```，将and后面的表达式去掉之后就正常了，那么就确定了问题是出在number 0与空字符串比较上。

我们都知道Mybatis中的表达式底层是通过OGNL实现的，OGNL是一种表达式语言，通过Java reflection对对象属性进行读写，支持对表达式进行求值，对不同类型的对象操作时会有隐式类型转换。

```menu.version != null and menu.version != ''"```这个表达式会被解析成一个AST，其中有两个ASTNotEq```menu.version != null```和```menu.version != ''"```，这两个ASTNotEq通过ASTAnd连接，第一个ASTNotEq没有问题，因为此时对象实例menu中的version字段是number 0，因此返回值为true，而第二个ASTNotEq在求值时，左侧menu.version的实际类型为number，而右侧是一个空字符串，求值时调用了OgnlOps.equal，这里存在四种情况的比较，1. 都是null 2. 引用相同 3. 左值右值同属于number类型，对于其他情况则通过OgnlOps.isEqual，其首先也是先比较引用是不是相同，其次是看左值和右值是不是相同类型的数组类型，否则的话先尝试调用Object.equals，最后才会调用OgnlOps.compareWithConversion：
```java
public static int compareWithConversion(Object v1, Object v2)
    {
        int result;
        if (v1 == v2) {
            result = 0;
        } else {
            int t1 = getNumericType(v1), t2 = getNumericType(v2), type = getNumericType(t1, t2, true);
            switch(type) {
            case BIGINT:
                result = bigIntValue(v1).compareTo(bigIntValue(v2));
                break;
            case BIGDEC:
                result = bigDecValue(v1).compareTo(bigDecValue(v2));
                break;
            case NONNUMERIC:
                if ((t1 == NONNUMERIC) && (t2 == NONNUMERIC)) {
                    if ((v1 instanceof Comparable) && v1.getClass().isAssignableFrom(v2.getClass())) {
                        result = ((Comparable) v1).compareTo(v2);
                        break;
                    } else {
                        throw new IllegalArgumentException("invalid comparison: " + v1.getClass().getName() + " and "
                                + v2.getClass().getName());
                    }
                }
                // else fall through
            case FLOAT:
            case DOUBLE:
                double dv1 = doubleValue(v1),
                dv2 = doubleValue(v2);
                return (dv1 == dv2) ? 0 : ((dv1 < dv2) ? -1 : 1);
            default:
                long lv1 = longValue(v1),
                lv2 = longValue(v2);
                return (lv1 == lv2) ? 0 : ((lv1 < lv2) ? -1 : 1);
            }
        }
        return result;
    }
```
代码看起来有点杂乱，不过逻辑并不复杂，简单来说：
1. 若左值和右值都为number类型，则按照number优先级转换并比较值；
2. 若左值和右值都不为number类型，则尝试通过Comparable.compareTo进行比较（如果都实现了Comparable的话），否则会抛IllegalArgumentException；
3. 若左值和右值有一个是number类型，则隐式转换成double进行值比较。

问题就出在3，空字符串在隐式转换成double的时候会变成0.0D，意味着```menu.version != ''"```表达式求值为false，因此dynamic SQL中的if表达式为false，导致了前面所说的bug。

### 后记
其实这篇文章中的规则在[OGNL官方文档](https://commons.apache.org/proper/commons-ognl/language-guide.html)的附录部分有介绍：
> Equality is tested for as follows. If either value is null, they are equal if and only if both are null. If they are the same object or the equals() method says they are equal, they are equal. If they are both Numbers, they are equal if their values as double-precision floating point numbers are equal. Otherwise, they are not equal. These rules make numbers compare equal more readily than they would normally, if just using the equals method.