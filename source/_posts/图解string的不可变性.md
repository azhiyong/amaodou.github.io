---
title: 图解Java String的不可变性
date: 2020-05-09 11:07:36
tags:
---
> 翻译自 [What is string immutability](http://www.programcreek.com/2009/02/diagram-to-show-java-strings-immutability/)

1. 定义字符串

   ```java
   String s = "abcd";
   ```

   s 存储了字符串对象"abcd"的引用，也可以理解为 s 指向字符串对象"abcd"。

   ![reference new string](/images/reference_new_string.jpeg)

2. 把一个 String 变量赋值给另一个 String 变量

   ```java
   String s2 = s;
   ```

   s2 也存储了字符串对象"abcd"的引用，s 和 s2 都指向字符串对象"abcd"。

   ![reference string](/images/reference_string.jpeg)

3. 字符串连接

   ```java
   s = s.concat("ef");
   ```

   这时 s 存储了一个新的字符串对象的引用，s 指向新的字符串对象"abcdef"。

   ![update reference](/images/update_reference.jpeg)

总结：

字符串一旦在内存（堆）中创建就不能改变。需要注意的是，String 对象的所有方法都不会改变字符串本身，而是返回一个新的字符串对象。

如果需要可变的字符串，则需要使用 StringBuffer 或 StringBuilder。否则每次创建新的字符串对象，将导致大量的时间耗费在垃圾回收（Garbage Collection）上。

```java
StringBuilder mutableString = new StringBuilder();
mutableString.append("abcd");
mutableString.append("ef");
mutableString.toString();
```
