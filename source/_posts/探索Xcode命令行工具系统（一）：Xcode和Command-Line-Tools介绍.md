---
title: 探索Xcode命令行工具系统（一）：Xcode和Command Line Tools介绍
date: 2021-08-06 14:38:57
tags:
---

本来标题想用「探索Xcode命令行生态」的，想想“生态”一词用在这里略显浮夸，就改成了“系统”（还有个原因，应该是受到了最近在看的[“系统之美”](https://book.douban.com/subject/11528220/)这本书的影响）。加上“工具”一词，是为了与"Command Line Tools"一词对译。

Xcode是苹果提供的用于开发运行于macOS, iPhone, iPad, Apple Watch, Apple TV等硬件平台上的软件的开发工具集。它不仅是Xcode IDE，也包括了iOS模拟器、iOS真机调试软件、各硬件平台SDK、Command Line Tools等等所有开发必须的软件。

Command Line Tools的全称是 Command Line Tools for Xcode x.x.x，其中x.x.x是Xcode的版本。Command Line Tools包含有LLVM toolchain、git、macOS SDK等软件，是开发的必需品。它可以随Xcode分发，也可以单独下载和使用（即不需要安装Xcode）。可能是出于减小包大小和简化发版的原因，最近的Xcode.app已不在自带Command Line Tools。开发者在第一次打开Xcode时，会跳出安装Command Line Tools的提示框。
其他安装Command Line Tools的方法还有：
* `sudo xcode-select --install`
* 单独下载并安装
* 在Xcode GUI中安装

小结：Xcode和Command Line Tools的关系是：Command Line Tools是Xcode的一部分，但是完全可以单独使用。比如你可以在mac构建机上不安装Xcode APP，只安装Command Line Tools。

为了减少大家每一篇的阅读时间，我把这个系列分成了5篇（暂定）。在下一篇文章里，我将通过探索`git -h`命令运行的例子，来正式开启我们这个系列的学习。
![-w1302](https://cdn.zlrs.site/mweb/2021/08/06/16282399579998.jpg)

Reference:
* [苹果开发者下载中心](https://developer.apple.com/download/all/)

