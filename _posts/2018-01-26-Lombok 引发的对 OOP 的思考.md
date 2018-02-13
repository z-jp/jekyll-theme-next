---
layout: post
title: Lombok 引发的对 OOP 的思考
category: 软件工程
tags: 软件工程
---
> OOP把对象作为程序的基本单元，一个对象包含了数据和操作数据的函数。

初学 Java Web 时，对 VO、DTO、DO 这些概念，对 Domian、Service、Dao 分层结构着迷不已。现在写再小一个 Web 项目也是这么分包，以为这就是面向对象设计的全部。 今天在 V2 上看到有[帖子](https://www.v2ex.com/t/425987)推荐 Lombok，便提了一个[困惑很久的问题](https://www.v2ex.com/t/425987#reply12)：代码生成插件如 Lombok 相比 IDE 的插入代码，优势在哪里？Java 的构造方法、getter/setter 确实尤其啰嗦，但 IDE 可以“一键”生成，而且 POJO 类通常也很少需要打开来查看（除了属性没有什么东西了...冗长的样板代码很少会是一个需要处理的问题。直到看到 [swim2sun](https://www.v2ex.com/member/swim2sun)@v2ex 的回答：

> 一开始看到那么人多喷 lombok 很诧异，很多人竟然只是因为要给 ide 装个插件就放弃了 lombok 但站在他们的角度想想，也许他们确实不太需要 lombok，如果你仍然用着 N 层架构，项目里就是一堆 service, dao, pojo 的话，那 getter/setter 方法全部集中在了 pojo 里，这种情况下 lombok 的意义确实不大，因为你根本不会在意 pojo 里有哪些函数。 但是，如果你恰好了解领域驱动设计（ DDD ），你不满足于贫血模型，开始实践充血模型，那么你的实体类里不但会有 getter/setter，还会有很多业务方法。这个时候你就会觉得那些 getter/setter 很多余、能自动生成的代码最好应该是在编译的时候生成，这是你发现 lombok 是个大救星。当然 lombok 的功能不只是生成 getter/setter，安装插件这个小小的麻烦与它带来的收益相比根本不算什么。

`贫血模型`，曾经看见但完全无法理解而很快被我遗忘的名词，指的是对象中仅有属性没有操作。而这样的 POJO 在 Java Web 项目中无处不在。回到文章开头，面向对象设计的实质是什么，是对象拥有状态和行为。贫血的 POJO 只封装了数据，作为数据的载体在Service、View各层之间传递，这不正是面向过程的设计吗？ 那么，一个彻底的面向对象设计是怎样的？我并不知道。终究还是代码敲得太少，吃不消软件过程的一套方法论。