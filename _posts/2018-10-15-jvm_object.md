---
layout: post
title: "JVM中对象的内存分布"
# subtitle: "This is the post subtitle."
comments: true
date:   2018-10-15 13:57:15 +0800
categories: 'cn'
---

JVM中对象的由object header+object instance+padding组成，而object header又由mark word以及klass pointer组成，其中klass pointer和padding是非必需的，后面将会由详细的解释。

### Mark word
在32位系统中占用4个byte，64位系统中占用8个byte，这里列出关键部分，详情请参考[markOop](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp)：
```zsh
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```

### Klass pointer
上文有提到klass pointer并不是object header中必需的部分，原因在于JVM采用何种方式实现对实例的引用，引用方式主要由两种，一种是通过handler，另一种是direct pointer，Hotspot中采用后者，因此Hotspot中的klass pointer是必需的。《深入理解Java虚拟机》中$2.3.3中有详细介绍这两者的区别及优缺点。

### Padding
字节填充部分同样也不是object header中必需的部分，Hotspot只规定类对象的内存起始地址必须是8bytes的整数倍，因此只有mark word+klass pointer+object instance的字节数不是8bytes的整数倍时才会有padding。

### Example
这里我用了java.lang.Long作为测试类，工具位(JOL)[http://openjdk.java.net/projects/code-tools/jol/]。
```zsh
*:Desktop *$ java -jar jol-cli-0.9-full.jar internals java.lang.Long
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# WARNING | Compressed references base/shifts are guessed by the experiment!
# WARNING | Therefore, computed addresses are just guesses, and ARE NOT RELIABLE.
# WARNING | Make sure to attach Serviceability Agent to get the reliable addresses.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

Instantiated the sample instance via public java.lang.Long(long)

java.lang.Long object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           b9 25 00 f8 (10111001 00100101 00000000 11111000) (-134208071)
     12     4        (alignment/padding gap)                  
     16     8   long Long.value                                0
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```
我本地的机器为64位系统，测试结果显示，java.lang.Long占用24bytes，object header占用12bytes，其中8bytes为mark word，4bytes为klass pointer（这里出现了[指针压缩](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)），padding为4bytes。