---
title: SOLID
date: 2019-12-24 10:01:16
tags:
- 设计原则
category:
- coding
---

SOLID 是一组鼎鼎大名的设计原则，应用这些原则是为了组织好**类和模块**，应用好 SOLID 原则是良好架构的基础。

<!--more-->

## SRP：单一职责原则

> Every class or module in a program should have responsibility for just a single piece of that program's functionality.

简单来说，这个原则说的是，每个类（或模块）只负责一个功能，甚至说一个函数也应该设计得尽量简短，一个函数只完成一个功能。这个原则看起来很简单，如果一个类过于庞大，方法数量过多；或者是一个函数行数过多，很可能就违背了 SRP。但是如果把类拆得太细了，会影响内聚性，不利于维护。所以要用好这个原则，需要经验的积累和仔细权衡取舍。

## OCP：开闭原则

> The software entities (classes or methods) should be open for extension but closed for modification.

这一条原则就相对没那么直观了。换句话来说，就是要在代码修改量尽可能少的情况下，扩展功能。怎么做？其中一点是依赖方向要组织好。如果 A 组件不想被 B 组件上的修改影响，那么就应该让 B 组件依赖于 A 组件。而且这种依赖还必须是单向的。举个简单的例子，如果 A 组件是发送短信的模块，B 组件是生成短信内容的模块，那么应该让 B 组件依赖 A 组件，而 A 组件则对 B 组件一无所知。

另外一点是要对系统进行适当的抽象化设计。还是发短信的例子，例如 A 组件是用阿里云发短信的模块，B 组件是用百度云发短信的模块，C 组件是生成短信内容的模块。那么就让 A 和 B 实现同一个接口或继承同一个抽象类，而 C 组件则依赖于抽象类，而不要依赖具体的实现（A 或 B组件）。

## LSP：里氏替换原则

> If S is a subtype of T, then objects of type T in a program may be replaced with objects of type S without altering any of the desirable properties of that program

这个是从语法上给出来的定义，理解起来也非常简单。但是它还有更严格的语义方面的定义。从语义上，子类要实现与父类相同的功能。更严格的是，子类不能违背父类对输入、输出和异常，以及代码注释中的约定。

## ISP：接口隔离原则

> Clients should not be forced to depend on methods that they do not use. Interfaces should belong to clients, not to libraries or hierarchies.

这一条原则相对抽象一点。核心思想就是，不要引入或依赖不相关的东西。假如这个「接口」是指前端和后端之间的 API 接口，那么这个接口应该做得足够小，聚焦一个功能；如果这个「接口」是指编程语法上的接口，那么就把接口中声明的方法做得尽量少；甚至说如果「接口」是指一个函数，那这个函数不要既实现这个功能，又实现那个功能（跟 SRP 有点像）。

## DIP：依赖反转原则

> High level modules should not depend on low level modules; both should depend on abstractions. Abstractions should not depend on details.

换句话来说，就是多引用抽象类型，而非具体实现。但是这条原则并不是绝对的。例如 A 模块要依赖于 B 模块，但是很确定 B 模块并不会频繁发生变化，或者很长时间内都不会变化，就没必要过度设计，依赖于抽象了，直接依赖于具体实现即可。

《架构整洁之道》中给出了一些具体的守则：
- 多使用抽象接口
- 不要在具体实现类上创建衍生类（少用继承，继承关系是依赖关系中最强的）
- 不要重写（override）包含具体实现的函数（跟上一条差不多意思）
- 避免在代码中写入与任何具体实现相关的名字，或者其他容易变动的事物的名字（这条很容易忽视）

## 总结

以上这 5 条原则的定义是死的，假如对它们的定义都忘光了也没关系，关键是理解好背后的思想：
- 一定要分析清楚对于当前的系统，哪些部分是经常变动的，哪些部分是不易变动的，哪些部分是需要经常复用的
- 每个组件的大小要设计好，不要承担太多的职责
- 要知道抽象层的意义，用好抽象类和接口

最后的最后，如果这 5 条原则感觉用不上，意味着平时写代码还是停留于面向过程式的代码，并不是面向对象式的代码 >.<

---

参考资料：
-  Robert C. Martin《架构整洁之道》第7-11章
