---
layout: post
title: "线上问题实战：NoSuchPropertyException——null到底有多坑"
# subtitle: "This is the post subtitle."
comments: true
date:   2018-12-11 11:45:33 +0800
background: '/assets/images/bg-index.jpg'
categories: 'cn'
---

## 问题起因：
收到线上告警，日志提示某条mapper中配置的SQL遇到org.apache.ibatis.ognl.NoSuchPropertyException

## 问题分析：
该SQL非最近添加，已经在master很长一段时间，并且分析了对应的PO发现字段确实存在，故排除是业务代码的bug。
尝试Google了一把exception，运气不错，首页结果里面就有mybatis的[issue-623]，description与我们线上遇到的问题一致，issue中的comment提到mybatis-3.3.1及3.4.0中引用的OGNL版本为3.1.2，其OgnlRuntime.getMethodValue在并发情况下可能会抛exception，准确的说是OgnlRuntime.getMethodValue调用了OgnlRuntime.getGetMethod，这个方法存在并发性问题，可能会错误的返回null，而OgnlRuntime.getMethodValue发现OgnlRuntime.getGetMethod返回null之后会调用OgnlRuntime.getReadMethod，巧的是这个方法在3.1.2的实现有bug导致永远返回null（很有意思，感兴趣的可以自己看下OgnlRuntime.getReadMethod的实现，这里就不做分析了），于是本来存在的property没有被找到，从而导致了NoSuchPropertyException。

### 进一步分析
#### 时序图（偷个懒没画返回）
![image](https://user-images.githubusercontent.com/3426457/49803565-24886480-fd8b-11e8-926c-77705e9e72db.png)
#### OgnlRuntime.getGetMethod
代码如下：
```java
public static Method getGetMethod(OgnlContext context, Class targetClass, String propertyName)
            throws IntrospectionException, OgnlException
    {
        // Cache is a map in two levels, so we provide two keys (see comments in ClassPropertyMethodCache below)
        Method method = cacheGetMethod.get(targetClass, propertyName);
        if (method != null)
            return method;

        // By checking key existence now and not before calling 'get', we will save a map resolution 90% of the times
        if (cacheGetMethod.containsKey(targetClass, propertyName))
            return null;

        method = _getGetMethod(context, targetClass, propertyName); // will be null if not found - will cache it anyway
        cacheGetMethod.put(targetClass, propertyName, method);

        return method;
    }
```
个人觉得问题的关键还是在于代码中对null的模糊定义导致写出来的代码逻辑难以自洽，比如在上面的代码中null既可以代表GetMethod还没有被获取过，也可以代表GetMethod不存在，如果改成下面的代码，理论上不仅可以保证线程安全性，也使得代码更易读而避免无用的注释。
```java
private static final OgnlMethod NOT_FOUND_METHOD = new OgnlMethod(null);
private static Object MONITOR = new Object();

public static Method getGetMethod(OgnlContext context, Class targetClass, String propertyName)
            throws IntrospectionException, OgnlException
    {
        OgnlMethod method = cacheGetMethod.get(targetClass, propertyName);
        if (method == null) {
            synchronized (MONITOR) {
                if (method == null) {
                    method = _getGetMethod(context, targetClass, propertyName);
                    cacheGetMethod.put(targetClass, propertyName, method);
                }
            }
        }
        return method.getRealMethod();
    }

// use lombok
@Data
@AllArgsConstructor
class OgnlMethod {
    private final Method realMethod;
}
```

### 问题解决方案：
根据mybatis的release note，将版本从3.3.1更新到3.4.6解决了该问题。


#### PS：mybatis-3.4.6依赖的Ognl其实也没有修复这个并发问题，而是修复了OgnlRuntime.getReadMethod的问题从而间接的修复了这个问题。。。

#### PS：吐槽一下我们厂的日志系统因为一些不知道什么原因的bug没有给出完整的callstack，真的很烦