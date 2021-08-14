---
title: 误用单例模式造成的问题（以开发一个SDK为例）
date: 2021-08-14 18:59:01
tags:
---

这里以开发一个AppLog SDK的例子来说明误用单例模式会造成哪些问题。

## 需求分析
开发的第一步，是确定SDK的功能。我们假设AppLog SDK有以下功能：
* 埋点上报
* 本次启动未来得及上报的埋点，可以在下次启动时上报
* 能自动为以下事件做埋点：应用启动、进入前台、进入后台和应用退出

## 细分功能点和讨论软件开发面
* 埋点上报
    * 需要一个SDK主类，作为用户界面
* 本次启动未来得及上报的埋点，可以在下次启动时上报
    * 埋点数据要持久化到磁盘。需要数据库访问层（DAO）。
    * 需要埋点上报层，并支持不同上报策略（定时上报，手动触发上报等）
* 能自动为以下事件做埋点：应用启动、进入前台、进入后台和应用退出
    * 需要一个监听上述生命周期事件的对象

## 讨论实现：完全使用单例模式来实现
经过以上两步我们分析完了AppLog SDK的功能点，并分出了用户界面层、数据库访问层、上报服务层和应用生命周期埋点层。我简单花了一个层级图表示层间的关系。（这张图不能完全反映软件的真实关系，比如若用户有手动触发上报的需求，则用户页面也需要依赖上报服务的上报接口，从而用户界面画成“L形”会更合适。但是这并不妨碍我们接下来的分析。）

 ![我们划分出的软件层级](media/16289268309087/16289281989139.jpg)
 
 因为每个层最少使用一个对象就可以完成，所以我们很容易想到用每个层都用一个单例。可以定义以下单例：
 * [AppLogSDK sharedInstance]
 * [AppLogSDKImpl sharedInstance]
 * [AppLogDAO sharedInstance]
 * [AppLogReportService sharedInstance]
 * [AppLogAppLifeCycleTrack sharedInstance]
 
单例之间的通信就是层间通信。5个单例的相互通信实现了埋点数据的流转。这可以让我们的SDK正常工作。但是存在以下代价：
1. 单例具有“惯性”。因为后续的开发者会贴近之前代码的实现风格，所以仓库中的单例会随着时间越来越多。可能某个需求用单例实现是不够佳的，但是因为codebase中已经全是单例了，所以后续的开发者往往也会用单例来实现。
2. 难以看清类之间的依赖关系。这是因为单例具有全局名字空间，所以依赖一个单例的对象无需在其接口（成员变量）中声明，而只需要随用随取即可。为了确定依赖，我们不得不去看代码中具体的调用点。（特别是在Objective-C中，一个类的代码可能分布在多个文件，更是令人头疼）这也为静态分析、依赖分析工具的开发增加了难度。当然，作为软件的第一任开发者和依赖关系的最初设计者，你在一开始肯定了解这其中的关系，只是对于新人和N月后再看代码的你来说有难度。
3. 难以编写单元测试。单元测试简单的说就是先确定一个类所有重要的test case，通过mock掉它所有的依赖+并运行被测代码(CUT, Code Under Test)来确定这个类的代码有没有问题。但是由于单例是随用随取的，我们很难去mock CUT对单例的依赖。(在这里例子里，由于AppLogImpl直接依赖了AppLogDAO单例，test case就无从mock数据库依赖)这样我们就无法确定出错时是CUT的错误还是依赖的错误；甚至单测通过时也无法确定是否是“错错得对”。
4. 容易承担过多职责，造成类级别的职责混乱。由于众多单例之间交叉调用的数据流是非常难解读的，而需求紧急时没有足够的时间去了解整个codebase的数据流。这样新人做需求时就容易往已有的单例上添加其他职责，造成职责混乱。
5. 内存占用较高。单例具有静态生命周期，只有在进程退出时才会被析构。当不再需要某个单例时，这个单例也会一直存在，直到进程退出。
6. 较难找到单例的创建时间。要确定单例的创建时间，得先找到所有`sharedInstance`的调用点，然后在这些调用点中找到时间上最前的那个。当你要为单例注入外部依赖时，弄清这一点是很重要的，否则外部依赖的注入可能晚于单例第一个方法调用。
7. 当未来需求变化时有潜在的巨大改造成本。想象一下，当单例难以满足新增需求时，如需求要求有多个埋点渠道、上报渠道、数据库为了合规也得按渠道隔离；那么单例的设计就会存在很大的改造成本。
 
## 讨论实现：使用组合设计模式来实现
Wikipedia对组合设计模式（Composite pattern）的介绍如下：
> In software engineering, the composite pattern is a partitioning design pattern. The composite pattern describes a group of objects that are treated the same way as a single instance of the same type of object. The intent of a composite is to "compose" objects into tree structures to represent part-whole hierarchies. Implementing the composite pattern lets clients treat individual objects and compositions uniformly.

在功能上，SDK的各个功能是由各个子模块的功能所组成的。在程序结构上，SDK的主类对象也可以是由多个子模块的对象所组成。成员类型可以是Inteface或者抽象类。
```
class AppLogSDKImpl {
   // instance properties
   AppLogDAOInteface dao,
   AppLogReportServiceInteface reportService,
   AppLogAppLifeCycleTrackInteface lifeCycleTrack
}
```
![组合设计模式形成的树形成员关系图](media/16289268309087/16289382645371.jpg)

相比单例，这带来了以下好处
* 方便搞清类间依赖关系。只需要阅读类的接口代码，就能知道其依赖。相关工具的开发也因此简便。
* 方便单元测试。子模块的对象作为主类对象的成员，可以应用接口抽象、继承或者依赖注入，便于被测试用例mock。每个类都可以被轻松地剥离出应用，并在注入mock依赖后成为一个独立的子系统。
* 有助于维护单一职责模型。由于无法被当做全局变量使用，就不会有人出于偷懒，把一些职责挂载到其它对象上。
* 功能更强大，能更好地应对需求变更。

## 感想
熟悉面向对象不仅仅是熟悉面向对象语言的关键词和语法而已，更重要的是
* 在语言无关的层面熟悉面向对象的思想
* 了解设计模式和对象关系图（类成员依赖关系图、对象创建关系图、函数调用关系图）
* 多写代码，多看好的代码

## Reference

* [What are drawbacks or disadvantages of singleton pattern?](https://stackoverflow.com/questions/137975/what-are-drawbacks-or-disadvantages-of-singleton-pattern)
* [Singletons are Pathological Liars](https://testing.googleblog.com/2008/08/by-miko-hevery-so-you-join-new-project.html)