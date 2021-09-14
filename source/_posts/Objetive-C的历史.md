---
title: Objetive-C的历史
date: 2021-09-14 14:04:46
tags: Objetive-C 历史
---

> 本文写于2020年7月

## 大事年表

| 时间 |        事件 |
|--|--|
| 1981 |        Brad Cox和Tom Love受到SmallTalk的启发，编写了一个用于支持SmallTalk语法的预处理器。Cox开始思考软件系统的重用问题。 |
| 1983 |        Cox和Love合伙创办PPI，销售Objective-C和相关库（他们称之为Software IC） |
| 1986 |        Cox出版《Object-Oriented Programming, An Evolutionary Approach》，介绍OC。 |
| 1988 |        NeXT买下OC授权，并推动GCC支持OC编译，还开发了AppKit和FoundationKit，作为UI编程库和基础库。 |
| 1992 |        GNU支持Objective-C（GNUStep）i.e. 支持Linux平台。 |
| 1996 |        苹果收购NeXT，包括OC、InterfaceBuilder、ProjectBuilder。NeXTStep开发环境被重命名为Cocoa。 |
| 2006 |        Objective-C 2.0（之前是叫objc4）。随iOS4一起发布。新增ARC，属性，64位支持，Class extensions，Associated Objects，tagged pointer等特性。 |
| 2011 |        Xcode 4.0发布，采用LLVM作为默认编译器。 |
| 2016 |        微软发布WinObjC，为Windows提供Objective-C开发环境 |
## 早期时代
Objective-C 主要由 Stepstone 公司的布莱德·考克斯（Brad Cox）和 汤姆·洛夫（Tom Love） 在 1980 年代发明。

1981年 Brad Cox 和 Tom Love 还在 ITT 公司技术中心任职时，接触到了 SmallTalk语言。Cox 当时对软件设计和开发问题非常感兴趣，他很快地意识到 SmallTalk语言 在系统工程构建中具有无法估量的价值，但同时他和 Tom Love 也明白，目前 ITT 公司的电子通信工程相关技术中，C 语言被放在很重要的位置。

于是 Cox 撰写了一个 C 语言的预处理器，打算使 C 语言具备些许 Smalltalk 的本领。Cox 很快地实现了一个可用的 C 语言扩展，此即为 Objective-C语言的前身。到了 1983 年，Cox 与 Love 合伙成立了 Productivity Products International（PPI）公司，将 Objective-C 及其相关库商品化贩售，并在之后将公司改名为StepStone。1986年，Cox 出版了一本关于 Objective-C 的重要著作《Object-Oriented Programming, An Evolutionary Approach》，书内详述了 Objective-C 的种种设计理念。

## NextStep时代
1988年，斯蒂夫·乔布斯（Steve Jobs）离开苹果公司后成立了 NeXT Computer 公司，NeXT 公司买下 Objective-C 语言的授权，并扩展了著名的开源编译器GCC 使之支持 Objective-C 的编译，基于 Objective-C 开发了 AppKit 与 Foundation Kit 等库，作为 NeXTSTEP 的的用户界面与开发环境的基础。虽然 NeXT 工作站后来在市场上失败了，但 NeXT 上的软件工具却在业界中被广泛赞扬。这促使 NeXT 公司放弃硬件业务，转型为销售NeXTStep（以及OpenStep）平台为主的软件公司。

1992年，自由软件基金会的 GNU 开发环境增加了对 Objective-C 的支持。1994年，NeXT Computer公司和Sun Microsystem联合发布了一个针对 NEXTSTEP 系统的标准典范，名为 OPENSTEP。OPENSTEP 在自由软件基金会的实现名称为 GNUstep。


## Apple时代
1996年12月20日，苹果公司宣布收购 NeXT Software 公司，NEXTSTEP/OPENSTEP环境成为苹果操作系统下一个主要发行版本OS X的基础。这个开发环境的版本被苹果公司称为Cocoa。

2005年，苹果电脑雇用了克里斯·拉特纳及LLVM开发团队[4]，clang及LLVM成为苹果公司在GCC之外的新编译器选择，在 Xcode 4.0之后均采用 LLVM 作为默认的编译器。最新的 Modern Objective-C 特性也都率先在 Clang 上实现。

2016年，微软发布WinObjC，作为Windows-iOS Bridge的基础，为Windows提供了Objective-C开发环境。
