# 什么是空指针异常，如何修正它

- 原文链接：[What is a NullPointerException, and how do I fix it?](https://stackoverflow.com/questions/218384/what-is-a-nullpointerexception-and-how-do-i-fix-it)

<!-- TOC -->

- [An Example](#an-example)
- [What is NullPointerException](#what-is-nullpointerexception)
- [how do we fix it](#how-do-we-fix-it)
    - [Identify the null values](#identify-the-null-values)
    - [Trace where these values come from](#trace-where-these-values-come-from)
    - [Trace where these values should be set](#trace-where-these-values-should-be-set)
    - [Other fixes](#other-fixes)
- [Best Practice](#best-practice)
- [Reference](#reference)

<!-- /TOC -->

---

## An Example

当声明一个引用变量时，实际上我们获得的是指向这个变量的指针。下面的代码声明了一个 `int` 类型的变量：

```java
int x;
x = 10;
```

变量 `x` 为 `int`，Java 会在声明时将其初始化为 0（在栈上）。当 `x = 10` 执行时，`10` 就会被写到变量 `x` 所指向的内存位置。

但是当声明一个引用类型的变量时就不一样了，看下面的代码：

```java
Integer num;
num = new Integer(10);
```

第一行声明了一个变量 `num`，但是它是一个引用类型，但是此时我们并未给他赋值，这时他什么都没有引用。

第二行创建了一个 `Integer` 类型的对象，并且把这个对象的引用赋给 `num`。此时我们就可以通过 `.` 来访问这个对象了。

`NullPointerException` 发生在声明了变量，但是没有创建对象。如果试图在创建对象前访问 `num` 变量，就会产生一个 `NullPointerException`。大多数时候编译器会发现这个问题，并给出提示；但是有时候你并没有直接去创建这个对象，而是直接使用。对于这种情况：

```java
public void doSomething(SomeObject obj){
    //do something to obj
}
```

这种情况下并没有直接创建 `obj` 对象，而是假设在方法调用以前已经创建好了。但是有时会用这种方式来调用这个方法

```java
doSomething(null);
```

这时 `obj` 就是 `null`。如果这个方法要对传进来的对象进行操作，那么这时抛出 `NullPointerException` 是比较合适的。因为这是程序员的责任并且程序员需要这个信息来做调试。

另外，如果这个方法并不一定要对这个对象进行操作，那么此时空指针可能是可以接受的。在这种情况下，你需要去检测空指针，并且在文档中给出说明。一个例子：

```java
/**
 * @param ojb An optional foo for _____. May be null, in which case the result will be _____.
 */
public void doSomething(SomeObject obj){
    if(obj != null){
        // do something
    } else {
        // do something else
    }
}
```


## What is NullPointerException

当应用程序试图在本应该使用对象的时候使用了 `null` 时，会抛出该异常。这些情况有：
- 试图调用 `null` 对象的实例方法。
- 访问或修改 `null` 对象的域。
- 当 `null` 对象的类型是数组时，试图获取其长度。
- 当 `null` 对象的类型是数组时，试图访问或修改其中元素。
- 主动抛出 `null` 对象时。

应用程序应该抛出 `NullPointerException` 类型的实例来暗示 `null` 对象的其他不合法使用。`NullPointerException` 也可以由虚拟机构造。

另外：
```java
SynchronizedStatement:
    synchronized( Expression ) Block
```

当 `Expression` 为 `null` 时，也会抛出该异常。

## how do we fix it

看一个例子：
```java
public class Printer {
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    public void print() {
        printString(name);
    }

    private void printString(String s) {
        System.out.println(s + " (" + s.length() + ")");
    }

    public static void main(String[] args) {
        Printer printer = new Printer();
        printer.print();
    }
}
```

### Identify the null values

第一步是精确定位哪个值导致这个异常。学会去读栈轨迹非常重要，他会告诉你这个异常是从哪里抛出的：

```java
Exception in thread "main" java.lang.NullPointerException
    at Printer.printString(Printer.java:13)
    at Printer.print(Printer.java:9)
    at Printer.main(Printer.java:19)
```

我们可以看到这个异常是在 13 行抛出的（在 `printString()` 中）。通过调试器或添加一些日志语句来检查哪个值是 `null`。这个地方我们发现 `s` 是 `null`, 并且调用了 `length()` 方法。当 `s.length()` 语句删除以后就不会再抛出这个异常了。

### Trace where these values come from

第二步是检查这个空值是从哪里传过来的。通过沿着方法的调用栈向上查找，我么可以看到 `s` 是在 `print()` 方法中调用 `printString(name)` 来传入的，但是 `this.name` 是 `null`。

### Trace where these values should be set

`this.name` 通过 `setName(String )` 方法来设置。但是我们发现这个方法根本没有调用。如果这个方法被调用了，确保这些方法调用的顺序是正确的，也就是 `setName()` 方法要在 `print()` 方法之前调用。

这些信息足够我们解决这个问题了：在 `printer.print()` 之前添加 `printer.setName()` 方法调用

### Other fixes

给变量一个默认值：

```java
private String name = "";
```

`print()` 或 `printString()` 方法对变量进行检测：

```java
printString((name == null) ? "" : name);
```

设计这个类的时候不允许 `name` 属性为 `null`:

```java
public class Printer {
    private final String name;

    public Printer(String name) {
        this.name = Objects.requireNonNull(name);
    }

    public void print() {
        printString(name);
    }

    private void printString(String s) {
        System.out.println(s + " (" + s.length() + ")");
    }

    public static void main(String[] args) {
        Printer printer = new Printer("123");
        printer.print();
    }
}
```

## Best Practice

1. 使用 `final` 修饰变量来强制初始化
1. 避免从方法中返回 `null`，例如返回空集合。
1. 使用注解 `@NotNull` 和 `Nullable`。
1. 快速失败并且使用 `assert` 来防止 `null` 对象在整个应用程序中传播。
1. 使用 `equals()` 方法时，已知的对象在前：`if("knownObject".equals(unknownObject))`
1. 使用 `valueOf()` 而不是 `toString()`
1. 使用工具类方法进行判断：`Objects.isNull(null);`
1. 对于那些并不由你创建的对象，总是检测 `null`。如果调用者传了 `null`，但是 `null` 不是一个合法的参数，那么应该抛出异常给调用者。简单的忽略不合法输入隐藏了代码中的问题，应该禁止这么做。

## Reference

1. [java.lang.NullPointerException -- Java™ Platform Standard Ed. 8](https://docs.oracle.com/javase/8/docs/api/)
