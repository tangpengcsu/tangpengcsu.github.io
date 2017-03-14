---
title: scala 类型系统
date: 2017-3-2 18:39:04
categories:
  - scala
  - 类型系统
tags:
  - scala
---

## upper bounds

> 上界

- 标识： <:

```scala
def pr(list : List[_ <: Any]) {
           list.foreach(print)
}
```

<!-- more -->

## lower bounds

> 下界

- 标识： >:

```scala
def append[T >: String] (buf : ListBuffer[T])  = {  
                buf.append( "hi")
}
```

## view bounds

- 标识 <%
- <% 的意思是 "view bounds" (视界)，它比 <: 适用的范围更广，除了所有的子类型，还允许隐式转换过去的类型
- 适用于 **方法与类** ，但是不适用于 trait（特征）

```scala
def method [A <% B](arglist): R = ...
```

## context bounds

## mutiple bounds


**不能同时有多个upper bounds 或 lower bounds，变通的方式是使用复合类型**

```scala
T <: A with B
T >: A with B
```

**可以同时有upper bounds 和 lower bounds**

```scala
T >: A <: B
```

这种情况 lower bounds 必须写在前边，upper bounds写在后边，位置不能反。同时A要符合B的子类型，A与B不能是两个无关的类型。

**可以同时有多个view bounds**

```scala
T <% A <% B
```

这种情况要求必须同时存在 T=>A的隐式转换，和T=>B的隐式转换。

```scala
scala> implicit def string2A(s:String) = new A
scala> implicit def string2B(s:String) = new B

scala> def foo[ T <% A <% B](x:T)  = println("OK")

scala> foo("test")
OK
```

**可以同时有多个context bounds**

```scala
T : A : B
```

这种情况要求必须同时存在A[T]类型的隐式值，和B[T]类型的隐式值。

```scala
class A[T];
class B[T];

implicit val a = new A[Int]
implicit val b = new B[Int]

def foo[ T : A : B ](i:T) = println("OK")

foo(2)
OK
```

## 协变、逆变

> 如果一个类型是协变或者逆变的，那么这个类型即为可变-variance类型，否则为不可变-invariance的。

在类型定义申明时，`+` 表示协变，`-` 表示逆变

### 协变：covariance

```scala
trait List[+T] // 在类型定义时(declaration-site)声明为协变
```

如 A 继承 B，便有 List[A] 继承 List[B]

### 逆变：contravariance

如 A 继承 B，便有 List[B] 继承 List[A]


要注意 variance 并不会被继承，父类声明为 variance，子类如果想要保持，仍需要声明:

```scala
trait A[+T]

class C[T] extends A[T]  // C是invariant的

class X; class Y extends X;

val t:C[X] = new C[Y]
<console>:11: error: type mismatch;
 found   : C[Y]
 required: C[X]
Note: Y <: X, but class C is invariant in type T.
You may wish to define T as +T instead. (SLS 4.5)
```

必须也对 C 声明为协变的才行：

```scala
class C[+T] extends A[T]

val t:C[X] = new C[Y]
t: C[X] = C@6a079142
```

## => 在 scala 中的含义

1. 匿名函数定义， 左边是参数，右边是函数实现体：(x: Int)=>{}
2. 函数类型的声明,左边是参数类型，右边是方法返回值类型：(Int)=>(Int)
3. By-name-parameter：f(p :=>Int)
4. case 语句中 case x => y
