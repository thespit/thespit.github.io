---
layout: post
title: "Java语言和虚拟机学习-虚拟机和字节码"
date: 2017-03-18
author: Thespit
---
# 背景介绍

### 什么是虚拟机

虚拟机是一种抽象的机器环境程序，用来让计算机能够运行特定的程序。本篇中说的是Java虚拟机（JVM），也就是运行Java编译之后的.class文件的环境程序。JVM包括了三个概念，规范，实现和实例。规范规定了JVM需要实现的内容，在规范的定义下，许多种不同的实现都能达到标准。实例指的是一个实际运行的JVM。

### 什么是字节码

JVM完全不知道任何Java语言的语法，它只能看懂一种特殊的二进制格式文件，也就是.class文件。一个.class文件内容包含了一系列JVM指令（或称之为bytecode字节码）、一个符号表（存储每一个变量对应的类型和作用域等信息），以及其他的一些辅助信息。

通过这个官方的解释，我们可以基本总结出，字节码就是JVM能看懂的语言。

我们使用javac命令将.java文件编译成.class文件，这也就是从人类能看懂的Java转换成JVM能看懂的字节码的过程。

### 为什么要学习字节码

学习或者使用Java语言的人，基本要求是能看懂Java。但从来没哪个Java程序员说，我用字节码写代码。既然如此，我们为什么还要自找麻烦去学习字节码呢？

前面说到了，Java是我们人类的编程语言，而字节码是JVM直接解释使用的语言。小时候我们都玩过一个游戏叫交头接耳，几个人一组，由裁判设定一个句子从第一个人开始以悄悄话的形式传下去，最后一个人听到的内容往往面目全非。同样的语言尚且如此，如果交头接耳中再穿插几个美国人，就算大家都懂英文的情况下，结果也会画美不看。

我并不是说Java编译器一定会曲解编程者的意思。在大部分情况下，编程者确实不需要知道字节码的存在。但是，因为Java编译器在生成字节码的时候会进行适当的优化和改写。如果我们的认知只停留在Java语言的层面上，对于JVM指令的执行顺序，性能优化等多方面的认识都会受限。假如一段程序需要极致的性能，那么查看生成的字节码相比只看代码，往往会更直接地告诉我们症结所在。

## 运行代码的原理

### 一个Java程序的诞生

一个Java程序的一生轨迹是如此的：

1. 程序员编写.java文件。
2. 调用javac命令（Eclipse，intellij等代码编辑器自带的编译功能也是如此）将.java文件编译成.class文件。
3. 调用java命令执行.class文件，JVM“解释”执行.class文件，真正地执行程序。

之所以解释加了引号，是因为这个解释并非完全是解释器一词中的解释。详见下一节。

### 编译器、解释器以及JIT

计算机执行一个程序有两种基本形式，编译执行和解释执行。

编译执行是在程序执行之前先花一些时间把人类编写代码变成计算机的机器码，比如从.c文件编译成.exe文件。

解释执行是不提前做任何准备，在程序执行的时候才一行一行的翻译代码成机器码。

编译的好处是执行起来速度快，因为执行的时候程序就已经变成机器码了，好似一个翻译官对着翻译好的稿子一口气念完。

解释的好处是程序写完就能直接运行，不需要先等待编译器翻译一遍，类似一个口译官在发言人说完一句话后立刻插入翻译一句。

乍看上去编译器是更好的选择，毕竟性能为王。用户才不关心你的.exe文件经历了多少的磨难才生成出来，他们只关心程序性能是否达标。但是即使除去每次修改都要重新编译太麻烦以外，编译器编译好的机器码往往只能在一种机器上运行，机器因为内核的不同，使用的机器码也是不同的。换了平台就需要重新编译，而且经常在不同环境下编译起来错误百出。开发流程繁琐，过于依赖平台，让一些传统编译执行语言（比如C，C++）在制作大型项目时麻烦频频。

想象一下，有这么一个纯C++编写的大型网络游戏，代码量巨大，编译一次需要1个小时。那么每次编译发布版本并提交测试的过程一定会重复这个步骤，修改5分钟，编译1小时。这种开发节奏几乎是成熟团队所无法忍受的。因此当下的基于C++引擎的游戏往往是使用lua这种解释型语言来编写主要的逻辑。底层引擎的代码规模大为减少，同时逻辑的修改也并不会导致引擎的重新编译。

解释执行语言使用起来就完全没有上述的痛苦，代码写完就能立刻执行。而且只要各个平台的电脑预先装好了解释器，程序可以更顺利地跨平台执行。但是他执行时一行一行翻译的特点，确实很慢。

于是JIT（Just In Time)出现了，理所应当地结合了两者的特点。

JIT理论上是一个编译器，但是他运行起来更像是一个解释器，因为不需要提前编译代码，他可以在运行时才将代码编译成机器指令，这也是Just In Time名字的来源，编译只在最刚刚好要用的时候进行。JIT相对普通编译器更加智能，因为他是运行时才执行的，因此相对于普通编译器，JIT能够获取程序运行时的实时信息，这方便了他进行编译优化。比如将执行频率很高的代码优化后存储起来。

### HotSpot

HotSpot是Oracle当前使用的Java虚拟机版本，对于HotSpot，来自Oracle的介绍是这样的：

Java HotSpot虚拟机是Java SE平台的核心组件。它实现了Java虚拟机规范，也同时作为Java运行时环境的一个共享库。作为Java字节码的运行引擎，它提供了Java运行时的一些功能，例如在各种操作系统和机器架构上实现多线程，对象同步锁功能。它包涵了一个动态编译器，适时将需要的Java字节码编译优化为机器指令，同时高效地使用垃圾收集器管理Java堆内存，优化以同时达到低暂停和高吞吐。它提供了调优、监控的数据，Debug工具和其他一些应用。

HotSpot的JIT原理是，先使用解释器解释执行程序。当一段代码多次执行后，虚拟机使用编译器将这段代码编译成机器码存储起来，以后再次调用就会免去解释的时间。

## 字节码语法

### 指令和操作符

指令就是一行操作语句，一个.class文件是由一系列的指令组成的。

指令可以分解为两部分，操作符和参数。比如goto就是一个操作符，它将程序执行指针跳转到指定的行。实际代码中，goto后续的部分就是参数，比如goto 8，就是跳转到第8行。

JVM的操作符实际是一个字节的16进制数，我们看到的类似iadd，bipush这样的符号其实是助记符，也就是让人可以看懂的辅助符号。比如iadd实际是0x60，ladd实际是0x61。这也就是说JVM指令从0x00到0xff最多有256个。

操作符的格式通常是Txxx_y的形式，T表示这个操作符能操作的参数的类型，xxx表示实际的操作名称。y如果有的话，有时表示这个操作符操作的寄存器的序号，有时表示这个操作符操作的默认参数值。比如iadd的意思就是针对整型（int）进行加（add）的操作。lstore_0表示将一个长整型（long）参数存储到第0个本地变量中。lconst_0表示把长整数0压入到操作数栈中。一些特例，比如goto并没有参数类型的提示，那么表示这个操作符不针对指定类型的参数操作。

前面说过了JVM最多只有256个操作符，如果一个操作要针对每种参数类型，都设定一个操作符，比如add就可能有iadd（int），fadd（float），ladd（long），badd（byte），sadd（short）等等很多个变种。这很明显256的上限就不够用了。因此很多操作只定义了最必要的操作符，通常是针对int，float，long，double四种类型，类似boolean，byte，char，short的数据一般使用int的操作符一并处理了。

### 操作数栈

操作数栈，首先是一个堆栈，堆栈是什么就不多解释了，一种后进先出的只有一个入口的容器。

操作数栈顾名思义存储的是操作数。而操作数，简单来说，是机器指令操作的变量。

譬如，假设add 2 1的意思是求2加1的和，那么2和1都是add的操作数，同时还有一个隐藏的操作数就是这个指令的结果3，3也是一个操作数。

计算机在执行指令的时候，存储这些操作数的地方就是操作数栈。因此执行add 2 1指令可以分解为以下几个步骤：

1. 将2压入操作数栈。
2. 将1压入操作数栈。
3. 执行add命令，这个命令自动从操作数栈中推出两个数字，并求和。
4. 将结果压入操作数栈中。

## 实例分析

### 常用的指令例子

bipush x // 将byte类型参数x压入到操作数栈中，并存储成整数。

iconst_x // 将整数x压入到操作数栈中，有0，1，2，3几个变种，因为不需要参数，因此相当于bipush针对这几个整数的简化形式。

istore_x // 从操作数栈中取出整数参数，存储到第x个本地变量中。

iload_x // 从第x个本地变量中读取整数参数，压入到操作数栈中。

if_icmplt x // 从操作数栈顶弹出两个整数，如果第二个整数小于第一个（lt，less than），那么	就跳转到第x行。需要注意顺序，先后从栈中弹出两个数，第二个弹出的数放在比较符的左边，先弹出的数放在比较符的右边。

字节码操作符是自己的一个参数栈的，比如if_icmplt 5实际上是除了参数5以外，还会从参数栈中pop两个之前存储的数字出来，如果第一个比第二个大，就跳转到第五行。比如之前的iload_0和bipush。

### 实际代码

JVM提供了一个非常方便的命令，javap，可以帮助我们查看.class文件对应的字节码。

简单的使用方式是：

javap -c abc.class

接下来我们来看一个实际的例子，体会一下代码细小的改动之后，对应的字节码会有如何的应对。

```java
public void test() {

    int i = 13;

    i = i + 1;

}
```
转换成：

``` 
public void test();

  Code:

     0: bipush        13

     2: istore_1

     3: iinc          1, 1

     6: return
```

bipush 13的意思是设置一个byte参数到操作数栈中，存储成int。

istore_1的意思是从操作数栈中获取一个int参数，并存储到1号本地变量中。

iinc 1,1 的意思是将1号本地变量中的值加1。



我们将代码稍微修改一点，将i的初始值变成1300，超过255的byte类型大小限制。

```java
public void test() {

    int i = 1300;

    i = i + 1;

}
```

转换成：

```
public void test();

  Code:

     0: sipush        1300

     2: istore_1

     3: iinc          1, 1

     6: return
```

相对于第一段代码，唯一变化的地方就是bipush变成了sipush，从指令名上我们就能看出，sipush的意思是将一个短整（short）类型的数字存储到操作数栈中，同时转换成int存储。



按照这个思路继续修改，i的初始值修改超过短整型的限制：

```java
public void test() {

    int i = 201010210;

    i = i + 1;

}
```

转换成：

```
public void test();

    Code:

       0: ldc           #15                 // int 201010210

       2: istore_1

       3: iinc          1, 1

       6: return

}
```

我们将i的初始值设置为了一个非常大的数字。当bipush和sipush都不适合存储一个数字的时候，JVM会使用ldc指令代替，ldc的意思是从常量池（constant pool）中读取一个常量（位置是#15）。可以想象，JVM会提前把数字201010210丢到常量池里。



继续修改i的类型为长整型：

```java
public void test() {

    long i = 21000000000L;

    i = i + 1;

}
```

转换成：

```
public void test();

    Code:

       0: ldc2_w        #15                 // long 21000000000l

       3: lstore_1

       4: lload_1

       5: lconst_1

       6: ladd

       7: lstore_1

       8: return
```


如果i变量是long型，那么生成的字节码就长了很多。主要是因为iinc操作符无法针对长整型数字进行自加操作。

ldc2_w #15从#15中读出一个数值压到操作数栈里。

lstore_1将操作数栈顶的值存储到1号本地变量里。

lload_1将1号本地变量里的值压到操作数栈里。

lconst_1将一个长整型的数字1压到操作数栈里。

ladd从操作数栈里推出两个长整型参数，相加，将结果压到操作数栈里。

lstore_1将操作数栈顶的参数存储到1号本地变量里。

其实基本逻辑没有变，但是相对于整型操作的自增操作符iinc，JVM对于长整型参数就几乎没有什么优化了。从Java文件中同样都是四行代码，转换成字节码之后，JVM实际执行的行数就有了天壤之别。