---
layout: post
title: Effective Java 笔记
date: 2017-2-27 12:39:04
tags:
  - JAVA
  - Effective Java
  - 笔记
categories:
  - JAVA
  - Effective Java
  - 笔记
---

## 1\. 考虑静态工厂方法代替构造器

### 优势:

1. 它们有名称。
2. 不必在每次调用它们的时候都创建一个新对象。
3. 它们可以返回原返回类型的任何子类型的对象。
4. 在创建参数化类型实例的时候，它们使代码变得更加简洁。

<!-- more -->

### 缺点：

1. 类如果含共有的或者受保护的构造器，就不能被子类化。
2. 它们与其他的静态方法实际上没有任何区别。

## 2\. 遇到多个构造器参数时要考虑用构建器

> 静态工厂和构造器有个共同的局限性：它们都不能很好的扩展到大量的可选参数。

### 重叠构造器

重叠构造器模式可行，但是当有血多参数的时候，客户端代码会很难编写，并且仍然较难阅读。

### JavaBeans 模式

在这种模式下，调用一个无参构造器来创建对象，然后调用 setter 方法来设置每个必要的参数，以及每个相关的可选参数。

这种模式弥补了重叠构造器模式的不足。

但是，JavaBeans 模式自身有着很严重的缺点。因为构造过程被分到了几个调用中，`在构造过程中 JavaBean 可能处于不一致的状态`。类无法仅仅通过校验构造器参数的有效性来保证一致性。同时，JavaBeans 模式也阻止了把类做成不可变的可能，这就需要程序员付出额外的努力来保证它的线程安全。

### Builder 模式

它既能保证重叠构造器模式那样的`安全性`，又能保证像 JavaBeans 模式那么好的`可读性`。

```Java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder{
       //必选参数
        private final int servingSize;
        private final int servings;
        //可选参数
        private int calories = 0;
        private int fat = 0;
        private int carbohydrate = 0;
        private int sodium = 0;

        public Builder(int servingSize, int servings){
            this.servingSize = servingSize;
            this.servings = servings;
        }
        public Builder calories(int val){
            calories = val;
            return this;
        }
        public Builder fat(int val ){
            fat = val;
            return this;
        }
        public Builder carbohydrate(int val){
            carbohydrate = val;
            return this;
        }
        public NutritionFacts build(){
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder){
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

不直接生成想要的对象，而是让客户端利用所有必要的参数调用构造器（或者静态工厂），得到一个 builder 对象。然后客户端在 builder 对象上调用类似于 setter 的方法，来设置每个相关的可选参数。最后，客户端调用无参的 build 方法来生成不可变的对象。这个 builder 是他构建的类的静态成员类。

- 与构造器相比，builder 的微略优势在于，builder 可以有多个可变（varargs）参数。
- builder 模式十分灵活，可以利用单个 builder 构建多个对象。
- 设置了参数的 builder 生成了一个很好的抽象方法。

总之，如果类的构造器或者静态工厂中具有多个参数，设计这种类时，Builder 模式就是中不错的选择，特别是当大多数参数都是可选的时候。与使用传统的重叠构造器模式相比，使用 Builder 模式的客户端代码将更容易阅读和编写，构建器也比 JavaBeans 更加安全。

## 4\. 通过私有构造器强化不可实例化的能力

对于自包含静态方法和静态域的。虽然名声不好，但是它们也确实特有用处。

这样的工具类不希望被实例化，实例化对它没有任何意义。

我们只需要让这个类包含`私有构造器`，它就不能被实例化了。

但是这样也有缺陷，它使得一个类不能被子类化。所有的构造器都必须显示或者隐式地调用超类构造器，在这种情况下，子类就没有可访问的超类构造器可调用了。

## 13\. 使类和成员的可访问性最小

### 封装/信息隐藏（information hiding/encapsulation）

### 概念

封装/信息隐藏：模块之间只通过它们的 API 进行通信，一个模块不需要知道其他模块的内部工作情况。

### 为什么要做信息隐藏

它可以有效地接触组成系统的各模块之间的耦合关系，使得这些模块可以独立地开发、测试、优化、使用、理解和修改。

### 规则

#### 尽可能地使每个类或者成员不被外界访问。

顶层的（非嵌套的）类和接口，只有两种可能的访问级别：包级私有的（package-private）和公有的（public）。

对于成员（域、方法、嵌套类和嵌套接口）有四种可能的访问级别：

- 私有的（private）
- 包级私有的（package-private）
- 受保护的（protected）
- 公有的（public）

对于共有成员，当访问级别从私有变成保护级别时，会大大增强可访问性。受保护的成员是类的导出的 API 的一部分，必须永远得到支持。导出的类受保护成员也代表了该类对于某个实现细节的公开承诺。`受保护的成员应该尽量少用`。

#### 实例域决不能是公有的

`包含公有可变域的类并不是线程安全的`。即使域是 final 引用，并且引用不可变的对象，当把这个域变成公有的时候，也就放弃了“切换到一种新的内部数据标识法”的灵活性。

同样的建议也是适用于静态域。只有一种例外情况：假设常量构成了类提供的整个抽象中的一部分，可以通过公有的静态 final 域来暴露这些常量。

注意，长度非零的数组总是可变的，所以，类具有公有的非静态 final 数组域，或者返回这种域的访问方法，这几乎总是错误的。如果类具有这样的域或者访问方法，客户端将能够修改数组中的内容。这是安全漏洞的一个常见根源。

总而言之，你应该始终尽可能地讲题可访问性。你再仔细地设计了一个最小的公有 API 之后，应该防止把任何散乱的类、接口和成员变成 API 的一部风。除了公有静态 final 域的特殊情况外，公有类都不应该包含公有域。并且要确保公有静态 final 域所应用的对象是不可变的。


## 15\. 在公有类中使用访问方法而非公有域

### 公有类不应该直接暴露数据域

- 如果类可以在它所在的包的外部进行访问，就提供访问方法，以保留将来改变该类的内部表示法的灵活性。如果公有类暴露了它的数据域，要想在将来改变其内部表示法是不可能的，因为共有类的客户端代码已经遍布各处了。
- 然而，如果类是包级私有的，或者是私有的嵌套类，直接暴露它的数据域并没有本职的错误。
- 让公有类直接暴露域虽然重来都不是种好办法，但是如果域是不可变的，这种危害就比较小一些。

总之，`公有类永远都不应该暴露可变的域`。虽然还是有问题，但是让公有类暴露不可变的域其危害比较小。但是，有时候会需要包级私有的或者私有的嵌套类来暴露域，无论这个类是可变的还是不可变的。

## 16\. 复合优于继承

继承是实现代码重用的有力手段。

### 继承打破了封装性

子类依赖于超类中特定的实现细节。超类的实现有可能会随着发行版本的不同而有所变化，如果真的发生了变化，子类可能会遭到破坏，即使它的代码完全没有改变。因而，`子类必须要跟着其超类的更新而演变`，除非超类是专门为了扩展而设计的，并且具有很好的文档说明。
