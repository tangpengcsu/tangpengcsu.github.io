---
layout: post
title: Java 笔记
date: 2017-2-27 12:39:04
tags:
  - JAVA
  - 笔记
categories:
  - JAVA
  - 笔记
---


## 协变、逆变与不变

** 逆变 ** 与 ** 协变 ** 用来 ` 描述类型转换（type transformation）后的继承 ` 关系，其定义：如果 X、Y 表示类型，f(⋅) 表示类型转换，≤ 表示继承关系（比如，A≤B 表示 A 是由 B 派生出来的子类）。

- f(⋅) 是协变（Covariant）的，当 X≤Y 时，f(X)≤f(Y) 成立；如`数组`，当然，泛型也可以通过通配符（extends、super）来实现协变与逆变
- f(⋅) 是逆变（Contravariant）的，当 X≤Y 时，f(Y)≤f(X) 成立
- f(⋅) 是不变（Invariant）的，当 X≤Y 时上述两个式子均不成立，即 f(X) 与 f(Y) 相互之间没有继承关系。如`泛型`

<!-- more -->
