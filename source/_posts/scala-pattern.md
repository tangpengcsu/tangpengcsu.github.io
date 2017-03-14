---
title: scala 模式匹配
date: 2017-3-14 18:39:04
categories:
  - scala
  - 模式匹配
tags:
  - scala
---

## 概念

scala 中的模式并不是指设计模式，它是数据结构上的模式，是用于描述一个结构的组成。

<!-- more -->

## 模式

### 常量模式(constant patterns)包含常量变量和常量字面量

```scala
scala> val site = "alibaba.com"
scala> site match { case "alibaba.com" => println("ok") }
scala> val ALIBABA="alibaba.com"
//注意这里常量必须以大写字母开头
scala> def foo(s:String) { s match { case ALIBABA => println("ok") } }
```

常量模式和普通的 if 比较两个对象是否相等(equals) 没有区别，并没有感觉到什么威力

### 变量模式(variable patterns)

确切的说单纯的变量模式没有匹配判断的过程，只是把传入的对象给起了一个新的变量名。

```scala
> site match { case whateverName => println(whateverName) }
```

上面把要匹配的 site 对象用 whateverName 变量名代替，所以它总会匹配成功。不过这里有个约定，对于变量，要求必须是以小写字母开头，否则会把它对待成一个常量变量，比如上面的 whateverName 如果写成 WhateverName 就会去找这个 WhateverName 的变量，如果找到则比较相等性，找不到则出错。

变量模式通常不会单独使用，而是在多种模式组合时使用，比如

```scala
List(1,2) match{ case List(x,2) => println(x) }
```

里面的 x 就是对匹配到的第一个元素用变量 x 标记。

### 通配符模式(wildcard patterns)

通配符用下划线表示："\_" ，可以理解成一个特殊的变量或占位符。

单纯的通配符模式通常在模式匹配的最后一行出现，case _ => 它可以匹配任何对象，用于处理所有其它匹配不成功的情况。
通配符模式也常和其他模式组合使用：

```scala
> List(1,2,3) match{ case List(_,_,3) => println("ok") }
```

上面的 List(\_,\_,3) 里用了2个通配符表示第一个和第二个元素，这2个元素可以是任意类型
通配符通常用于代表所不关心的部分，它不像变量模式可以后续的逻辑中使用这个变量。

### 构造器模式(constructor patterns)

这个是真正能体现模式匹配威力的一个模式！

我们来定义一个二叉树：

```scala
scala> :paste
//抽象节点
trait Node
//具体的节点实现，有两个子节点
case class TreeNode(v:String, left:Node, right:Node) extends Node
//Tree，构造参数是根节点
case class Tree(root:TreeNode)  
```

这样我们构造一个根节点含有2个子节点的数：

```scala
scala>val tree = Tree(TreeNode("root",TreeNode("left",null,null),TreeNode("right",null,null)))
```

如果我们期望一个树的构成是根节点的左子节点值为”left”，右子节点值为”right”并且右子节点没有子节点
那么可以用下面的方式匹配：

```scala
scala> tree.root match {
        case TreeNode(_, TreeNode("left",_,_), TreeNode("right",null,null)) =>
             println("bingo")
    }
```

只要一行代码就可以很清楚的描述，如果用java实现，是不是没这么直观呢？

### 类型模式(type patterns)

类型模式很简单，就是判断对象是否是某种类型：

```scala
scala> "hello" match { case _:String => println("ok") }
```

跟 isInstanceOf 判断类型的效果一样，需要注意的是scala匹配泛型时要注意，
比如

```scala
scala> def foo(a:Any) = a match {
            case a :List[String] => println("ok");
            case _ =>
        }
```

如果使用了泛型，它会被擦拭掉，如同java的做法，所以上面的 List[String] 里的String运行时并不能检测

foo(List("A")) 和 foo(List(2)) 都可以匹配成功。实际上上面的语句编译时就会给出警告，但并不出错。

通常对于泛型直接用通配符替代，上面的写为 case a : List[\_] => …

### 变量绑定模式 (variable binding patterns)

这个和前边的变量模式有什么不同？看一下代码就清楚了：

依然是上面的TreeNode，如果我们希望匹配到左边节点值为”left”就返回这个节点的话：

```scala
scala> tree.root match {
         case TreeNode(_, leftNode@TreeNode("left",_,_), _) => leftNode
        }
```

用@符号绑定 leftNode变量到匹配到的左节点上，只有匹配成功才会绑定

## 模式匹配方法

1. 面向对象的分解 (decomposition)
2. 访问器模式 (visitor)
3. 类型测试/类型造型 (type-test/type-cast)
4. typecase
5. 样本类 (case class)
6. 抽取器 (extractor)

### 样本类(case class)

本质上case class是个语法糖，对你的类构造参数增加了getter访问，还有toString, hashCode, equals 等方法；

最重要的是帮你实现了一个伴生对象，这个伴生对象里定义了apply 方法和 unapply 方法。

apply方法是用于在构造对象时，减少new关键字；而unapply方法则是为模式匹配所服务。

这两个方法可以看做两个相反的行为，apply是构造(工厂模式)，unapply是分解(解构模式)。

case class在暴露了它的构造方式，所以要注意应用场景：当我们想要把某个类型暴露给客户，但又想要隐藏其数据表征时不适宜。

### 抽取器(extrator)

抽取器是指定义了 unapply 方法的 object。在进行模式匹配的时候会调用该方法。

unapply 方法接受一个数据类型，返回另一数据类型，表示可以把入参的数据解构为返回的数据。

比如

```scala
class A
class B(val a:A)
object TT {
    def unapply(b:B) = Some(new A)
}
```

这样定义了抽取器 TT 后，看看模式匹配：

```scala
val b = new B(new A);
b match{ case TT(a) => println(a) }
```

直观上以为 要拿 b 和 TT 类型匹配，实际被翻译为

```scala
TT.unapply(b)  match{ case Some(…) => … }
```

它与上面的 case class 相比，相当于自己手动实现 unapply，这也带来了灵活性。

后续会专门介绍一下 extrator，这里先看一下 extractor 怎么实现 case class 无法实现的“表征独立”(representation independence)

比如我们想要暴露的类型为 A

```scala
//定义为抽象类型
trait A

//然后再实现一个具体的子类，有2个构造参数
class B (val p1:String, val p2:String) extends A

//定义一个抽取器
object MM{
    //抽取器中apply方法是可选的，这里是为了方便构造A的实例
    def apply(p1:String, p2:String) : A = new B(p1,p2);

    //把A分解为(String,String)
    def unapply(a:A) : Option[(String, String)] = {
        if (a.isInstanceOf[B]) {
         val b = a.asInstanceOf[B]
         return Some(b.p1, b.p2)
        }
        None
    }
}
```

这样客户只需要通过 MM(x,y) 来构造和模式匹配了。客户只需要和 MM 这个工厂/解构角色打交道，A 的实现怎么改变都不受影响。
