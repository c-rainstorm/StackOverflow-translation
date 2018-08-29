# 用 Java 创建内存泄漏

原文链接：[Creating a memory leak with Java](https://stackoverflow.com/questions/6470651/creating-a-memory-leak-with-java)


<!-- TOC -->

- [What is memory leak](#what-is-memory-leak)
- [How to create a memory leak](#how-to-create-a-memory-leak)
- [How to solve it](#how-to-solve-it)
- [Best practice](#best-practice)
- [Reference](#reference)

<!-- /TOC -->

---

## What is memory leak

内存泄漏是当一个计算机程序因为不正确的内存管理方式，导致不再使用的内存无法释放的一种资源泄漏情况。

在面向对象编程中，内存泄漏可能发生在一个对象在内存中，但是无法被正在运行时的代码访问，（也无法被虚拟机回收 -- 对于那些基于虚拟机技术的编程语言来说）的时候。

内存泄漏通常只能通过检查程序源代码来诊断。但是也有一些其他的工具来辅助我们定位问题，本文稍后的部分会有提到。

## How to create a memory leak

1. 创建一个长期运行的线程（或使用一个线程池）
1. 这个线程通过类加载器加载类
1. 分配一个大片段内存（`new byte[1000000]`）, 作为一个强引用保存到加载类的静态域，然后保存一个引用到 `ThreadLocal`
1. 线程清空所有到自定义类的引用或者类加载器

这个能产生内存泄漏的原因是 `ThreadLocal` 保留着一个到 `object` 的引用，这个 `object` 保留着对 `Class` 的引用，`Class` 保留着对 `ClassLoader` 的引用，`ClassLoader` 保留着对所有已加载类的引用。这些对象都无法被回收。

1. 常量静态域引用大对象

```java
class MemorableClass {
    static final ArrayList list = new ArrayList(100);
}
```

1. 使用 `String.intern()` 产生过长的字符串

```java
String str=readString(); // read lengthy string any source db,textbox/jsp etc..
// This will place the string in memory pool from which you cant remove
str.intern();
```

1. 未关闭的打开流/连接

```java
try {
    BufferedReader br = new BufferedReader(new FileReader(inputFile));

    Connection conn = ConnectionFactory.getConnection();
    ...
    ...
} catch (Exception e) {
    e.printStacktrace();
}
```

## How to solve it

## Best practice

## Reference

1. [Memory leak - Wikipedia](https://en.wikipedia.org/wiki/Memory_leak)
