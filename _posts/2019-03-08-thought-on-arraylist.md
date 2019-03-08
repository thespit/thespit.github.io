---
layout: default
title: "从ArrayList的优化中想到的"
date: 2019-03-08
author: Thespit
---
<h2>{{page.title}}</h2>

_{{page.author}}_

# 从ArrayList的优化中想到的

在JDK7中ArrayList有一个小的改动，使用延迟加载的思想，默认构造函数不再初始化生成一个大小为10的数组，而是将elementData先赋值为一个共享的空数组。

```java
package java.util;

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    private transient Object[] elementData;
    private int size;

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    // snipped
}
```
数组加载的时间延迟到了实际添加元素的时候。实际添加的时候，会去统一调用一个内部方法ensureCapacityInternal，判断elementData是否还等于默认值，如果还等于默认值就计算一个扩容量，并进行扩容。

```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

这个改动对于我们自己常写的那些随便new一个列表的懒代码来说，提升了很多性能。要知道之前的ArrayList总是默认就new Object[10]。

这本身不是一件大事，算法思路也很简单，但是触发了我的一些思考。

## 心态的差距

延迟初始化这种东西谁都会，我自己也经常用。ArrayList默认构造一个空间为10的数组，很多人都知道，没人说不好。我看见过不少次，作为常识记下来了，从没思考过是否合理，是否能改进。但现在他们又做了如此简单清晰的改进。让我感触很深，比起技术上的差距，心态上差距更大。从没想过自己有可能，就没可能了。

## 对于细节的优化总是存在

ArrayList已经是经历万代的数据结构了，居然还能迎来优化的修改。对于一个已经达到需求标准的代码，我们是否也能这样扣到每一个细节呢。反观项目里的那些随随便便的new，拷贝，通用数据结构的滥用。对比人家微雕般的手艺，我们自己就像是在工地开拖拉机铲土。事实上，一段频繁使用的代码，即使一行的改进带来的收益都会是巨大的。

## 基础知识和趋于平庸

刚毕业的时候通过刷题找工作，工作了之后往往发现算法、数据结构并没有人在乎。多年往后，又会拿起书本。常用的容器用多了就会产生它是唯一选择的错觉。面试的时候都说得出链表和数组的区别，可实际使用的时候谁还关心呢，还不都顺手写上new ArrayList<>()。确实对于一个大项目来说，一个临时变量用什么类型的容器来存储并不影响什么。但是对于性能的敏感性和自我要求也会在这些妥协中慢慢消失。所有的数据结构归为ArrayList和HashMap，所有的算法都是遍历查找。

没事的时候刷刷[leetcode](https://leetcode.com/problemset/all/)，看看框架源码，多读书。