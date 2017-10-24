# 为什么处理排序数组比未排序数组更快

- 原文链接：[Why is it faster to process a sorted array than an unsorted array?](https://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array)

<!-- TOC -->

- [题目描述](#题目描述)
- [答案整合](#答案整合)
    - [答案一](#答案一)
    - [答案二](#答案二)
- [最佳实践](#最佳实践)
- [参考](#参考)

<!-- /TOC -->

---

## 题目描述

这里有一段 C++ 代码，因为一些原因，排过序的数组执行效率比未排序数组快了近 6 倍。

```c++
#include <algorithm>
#include <ctime>
#include <iostream>

int main()
{
    // Generate data
    const unsigned arraySize = 32768;
    int data[arraySize];

    for (unsigned c = 0; c < arraySize; ++c)
        data[c] = std::rand() % 256;

    // !!! With this, the next loop runs faster
    std::sort(data, data + arraySize);

    // Test
    clock_t start = clock();
    long long sum = 0;

    for (unsigned i = 0; i < 100000; ++i)
    {
        // Primary loop
        for (unsigned c = 0; c < arraySize; ++c)
        {
            if (data[c] >= 128)
                sum += data[c];
        }
    }

    double elapsedTime = static_cast<double>(clock() - start) / CLOCKS_PER_SEC;

    std::cout << elapsedTime << std::endl;
    std::cout << "sum = " << sum << std::endl;
}
```

- 没有 `std::sort(data, data + arraySize);` 时，代码运行时间为 11.54s。
- 排过序以后代码运行时间是 1.93s。

一开始，我觉得这可能跟语言或编译器有关，所以我把这段代码改成了 Java。

```java
import java.util.Arrays;
import java.util.Random;

public class Main
{
    public static void main(String[] args)
    {
        // Generate data
        int arraySize = 32768;
        int data[] = new int[arraySize];

        Random rnd = new Random(0);
        for (int c = 0; c < arraySize; ++c)
            data[c] = rnd.nextInt() % 256;

        // !!! With this, the next loop runs faster
        Arrays.sort(data);

        // Test
        long start = System.nanoTime();
        long sum = 0;

        for (int i = 0; i < 100000; ++i)
        {
            // Primary loop
            for (int c = 0; c < arraySize; ++c)
            {
                if (data[c] >= 128)
                    sum += data[c];
            }
        }

        System.out.println((System.nanoTime() - start) / 1000000000.0);
        System.out.println("sum = " + sum);
    }
}
```

结果很类似，但是差距没有那么大。

- 其中发生了什么？
- 为什么排过序的数组更快？

## 答案整合

### 答案一

这是分支预测失败导致的。

- **什么是分支预测?**

考虑下面的铁轨链接处：

<a title="By Mecanismo (Own work) [CC BY-SA 3.0 (https://creativecommons.org/licenses/by-sa/3.0)], via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File%3AEntroncamento_do_Transpraia.JPG"><img width="512" alt="Entroncamento do Transpraia" src="https://upload.wikimedia.org/wikipedia/commons/thumb/c/cf/Entroncamento_do_Transpraia.JPG/512px-Entroncamento_do_Transpraia.JPG"/></a>

为了便于解释，我们假设回到了 1800 年，那时还没有远距离或无线电通信。

你是连接处的操作员并且你听到了一列火车正在驶来。你不知道这列火车要开往哪个方向。所以你叫停火车并询问驾驶员他想去哪个方向。然后你再切换到合适的轨道上。

_火车很重并且有很大的惯性，所以启动和停止都需要很长时间_

有更好的办法吗？猜测火车将要去哪个方向！
- 如果猜对了，它继续行驶。
- 如果猜错了，驾驶员会停下来，倒回到交汇点前，并要求你切换轨道。然后火车开始在另一条轨道上行驶。

如果每次都猜对的话，火车不需要停下来。
如果你经常猜错的话，火车会耗费大量的时间在停车、后退、启动上。

---

考虑下面的 `if` 语句：在处理器层次上，这里有一个分支指令：

![](https://i.stack.imgur.com/pyfwC.png)

假设你是处理器，并且你看到了一个分支。你不知道接下来会执行哪一个控制流。这时你该怎么办？暂停执行，并且等待之前的指令执行结束。然后你继续在正确的路径上执行。

_现代处理器都比较复杂并且拥有很长的流水线，所以启动(warm up)和暂停(slow down)的耗时都比较长_

有更好的办法吗？你可以猜测分支将要执行的方向！
- 如果猜对了，继续执行。
- 如果猜错了，你需要清空流水线，并回滚到分支执行之前。然后在另一个分支上执行。

如果每次都猜对，执行过程将永远不会停止
如果经常猜错的话，会耗费大量的时间在停止、回滚、重新启动上。

---

这就是分支预测。我承认这不是最恰当的类比，因为火车可以通过一些信号来说明要行使的方向。但是在计算机上，处理器直到最后一刻才知道他要执行的方向。

所以我们应该怎样猜测使得火车回退次数最少呢？查看通行的历史！如果火车 99% 从左侧过，那么你猜左侧；如果火车交替从两个方向行驶，那么你猜测未来火车也会交替；等等。

换句话说，你试图定义一个模式，并且遵循它。这大致就是一个分支预测器的工作原理。

大部分的应用程序都有表现良好的分支。所以现代的分支预测器的命中率超过 90%。但是当遇到不可预测的分支并且没有一个可辩别的模式时，分支预测其就显得无用了。

延伸阅读 ：["Branch predictor" article on Wikipedia](https://en.wikipedia.org/wiki/Branch_predictor)

---

如上所述，问题的根源就在于下面的 `if` 语句

```java
if (data[c] >= 128)
    sum += data[c];
```

注意数据是平均分布在 0 - 255 之间的，当数据排过序后，前半部分的数据都不会进入 `if` 语句中，后半部分都会进入 `if` 语句。

这对分支预测器就非常友好了，因为它可以连续进入相同的分支很多次。

快速预览：
```
T = branch taken
N = branch not taken

data[] = 0, 1, 2, 3, 4, ... 126, 127, 128, 129, 130, ... 250, 251, 252, ...
branch = N  N  N  N  N  ...   N    N    T    T    T  ...   T    T    T  ...

       = NNNNNNNNNNNN ... NNNNNNNTTTTTTTTT ... TTTTTTTTTT  (easy to predict)
```

可是当数据是完全随机时，分支预测器就变得无用了，因为它无法预测随机的数据。因此大概会有 50% 的几率误判。

```
data[] = 226, 185, 125, 158, 198, 144, 217, 79, 202, 118,  14, 150, 177, 182, 133, ...
branch =   T,   T,   N,   T,   T,   T,   T,  N,   T,   N,   N,   T,   T,   T,   N  ...

       = TTNTTTTNTNNTTTN ...   (completely random - hard to predict)
```

**我们应该怎么办？**

如果编译器无法将这种分支优化成条件转移指令(conditional move instruction（参考 CSAPP 2e 3.6.6 的翻译），更详细的说明请参考 [Conditioncal Move](https://www.cs.tufts.edu/comp/40/readings/amd-cmovcc.pdf))，如果你愿意为性能牺牲一些可读性的话，可以尝试一些其他的编码技巧。

替换：

```java
if (data[c] >= 128)
    sum += data[c];
```

用:

```java
int t = (data[c] - 128) >> 31;     //小于 128 的话， t = -1， ~t 为 0
sum += ~t & data[c];
```

这种方法使用位操作替换了分支。
(注意这种技巧和原来的 `if` 语句并不严格相同，但在这个场景下，它对所有的输入数据都是合法的)

测试环境： Core i7 920 @ 3.5 GHz

C++ - Visual Studio 2010 - x64 Release
```
//  Branch - Random
seconds = 11.777

//  Branch - Sorted
seconds = 2.352

//  Branchless - Random
seconds = 2.564

//  Branchless - Sorted
seconds = 2.587
```

Java - Netbeans 7.1.1 JDK 7 - x64

```
//  Branch - Random
seconds = 10.93293813

//  Branch - Sorted
seconds = 5.643797077

//  Branchless - Random
seconds = 3.113581453

//  Branchless - Sorted
seconds = 3.186068823
```

很明显的：
- 有分支：已排序和未排序的有很大不同
    - Java 中时间比例大概是 1：2， C++ 中是 1：6。造成这种差异的原因个人认为是 Java 的 JIT 的策略引起的，倍率被中间的实时编译时间拉小了。
- 去分支：已排序和未排序的性能基本相同
- 在 C++ 中，去分支的甚至比已排序有分支的还要慢一些。

一个惯例是在关键的循环中防止分支的走向依赖数据。

Update：

- 跟 Java 关系不太紧密，就不再翻了，有兴趣可以去读原文。

### 答案二

在 C++ 中我们可以通过使用以下替换方案来使用条件转移指令。

```c++
sum += data[c] >= 128 ? data[c] : 0;
```

**原因是 `... ? ... ? ...` 语句在编译器编译时会直接生成条件转移指令**

```c++
int max1(int a, int b) {
    if (a > b)
        return a;
    else
        return b;
}

int max2(int a, int b) {
    return a > b ? a : b;
}
```

在 x86-64 的机器上， `gcc -S` 生成以下汇编代码：

```assmbly
:max1
    movl    %edi, -4(%rbp)
    movl    %esi, -8(%rbp)
    movl    -4(%rbp), %eax
    cmpl    -8(%rbp), %eax
    jle     .L2                     ; 普通的比较跳转
    movl    -4(%rbp), %eax
    movl    %eax, -12(%rbp)
    jmp     .L4
.L2:
    movl    -8(%rbp), %eax
    movl    %eax, -12(%rbp)
.L4:
    movl    -12(%rbp), %eax
    leave
    ret

:max2
    movl    %edi, -4(%rbp)
    movl    %esi, -8(%rbp)
    movl    -4(%rbp), %eax
    cmpl    %eax, -8(%rbp)
    cmovge  -8(%rbp), %eax          ; cmovxx 条件转移指令
    leave
    ret
```

---

但是在 Java 中这种方法是不能用的

相同的函数代码：

```Java
public int withBranch(int a, int b) {
    if (a > b) {
        return a;
    } else {
        return b;
    }
}

public int conditionalMove(int a, int b) {
    return a > b ? a : b;
}
```

- OS：macOS Sierra version 10.12.6
- javac: javac 9

Java byte code:
```
> javap -c me.rainstorm.branch.WithBranch

public class me.rainstorm.branch.WithBranch {
  public me.rainstorm.branch.WithBranch();    // 默认无参构造函数
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public int withBranch(int, int);            // if-else
    Code:
       0: iload_1
       1: iload_2
       2: if_icmple     7
       5: iload_1
       6: ireturn
       7: iload_2
       8: ireturn

  public int conditionalMove(int, int);      // ...?...?...;
    Code:
       0: iload_1
       1: iload_2
       2: if_icmple     9
       5: iload_1
       6: goto          10
       9: iload_2
      10: ireturn
}
```

我们可以从字节码中看到，两种方式使用的都是 `if_icmple` 指令。他们在这一层次上是相同的，那么我们基本可以确定**机器码级别**也是一样的。

PS: 有兴趣的话可以使用 `64位 Linux 版本的 Java9 JDK` 中自带的 AOT 编译器 `jaotc` 生成机器码，对比看看是否一致。
- `注意：该工具目前只有这一个 JDK 版本中有，其他版本估计会在 Java10 中发布`

## 最佳实践

通过其他方法，如位操作将执行流程从分支执行转换为顺序执行。但是这种做法会牺牲一定的可读性，应该酌情使用。

## 参考

- 参考的其他链接忘记记录了，下不为例！
