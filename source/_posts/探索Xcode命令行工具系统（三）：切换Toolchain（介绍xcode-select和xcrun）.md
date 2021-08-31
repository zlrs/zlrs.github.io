---
title: 探索Xcode命令行工具系统（三）：切换Toolchain（介绍xcode-select和xcrun）
date: 2021-08-31 11:12:34
tags: macOS iOS
---


Mac上的每个Xcode APP都内置了一套工具链。工具链也可以单独下载，并部署在单独的文件夹中（如公司AppHealth团队维护的toolchain）。因此，一台Mac上可能存在多套工具链。
> 注：下面会混用toolchain和Command Line Tools两种说法。Command Line Tools包含了toolchain。

为了便于Toolchain的切换，苹果做了如下事情：
1. 提供“桩命令”. 具体什么是"桩命令"，可以参考**探索Xcode命令行工具系统（二）：git命令的调用过程**。 举个例子，`/usr/bin/git`和`/usr/bin/clang`都是桩命令，对他们的调用实际上是调用到当前toolchain中的对应命令。
2. 提供`xcode-select`命令和`DEVELOPER_DIR`环境变量，用于指定当前toolchain
3. 提供`xcrun`，用于运行当前toolchain中的命令或者打印该命令的路径。

## xcode-select介绍
这里简单介绍下参数和用法，具体用法可以看命令的manpage。
1. `-p`参数：打印当前toolchain路径。
2. `-s`参数：指定系统级toolchain路径。需要`sudo`权限。将影响本机上所有的用户。
3. `DEVELOPER_DIR`环境变量：用于指定当前环境的toolchain路径。可以覆盖系统级别的toolchain路径。

toolchain的路径一般在Xcode APP中，如`/Application/Xcode.app/Contents/Developer`（也可以写成`/Application/Xcode.app`，`xcode-select`会自动添加后面的部分）. 或者单独存在。

开发者也可以在Xcode IDE中切换系统级的toolchain（Command Line Tools）. 切换时需要指纹验证来获取sudo权限。其底层应该也是通过调用`xcode-select`实现。
![-w942](https://cdn.zlrs.site/mweb/2021/08/31/16285887284120.jpg)

**配合桩命令和`xcode-select`，就可以方便地切换toolchain。**
举个例子：假设我们在编译脚本中存在这样一行
```
xcodebuild xxx.c -xxxx  # 参数省略
```
一般来说，`xcodebuild`会被解析到桩命令`/usr/bin/xcodebuild`，桩命令再调到当前toolchain的`xcodebuild`。

问题：如果PATH头部存在其它Toolchain目录，则`xcodebuild`不会解析到桩命令。
如对于以下Path：
```shell
PATH=”/Applications/Xcode12.3.app/Contents/Developer/usr/bin:${PATH}”
```
`xcodebuild`被会解析到`/Applications/Xcode12.3.app/Contents/Developer/usr/bin/xcodebuild`，而不是桩命令，从而无法随toolchain的切换而改变路径。

## xcrun 介绍
这里简单介绍下参数和用法，具体用法可以看命令的manpage。
1. 执行当前toolchain的某个命令，如
```
xcrun git status
```
2. 查找命令的路径
```
zhuyuanqing@C02D70XMMD6R ~/B/tools> xcrun --find clang
/Applications/Xcode12.3.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang
zhuyuanqing@C02D70XMMD6R ~/B/tools> xcrun --find git
/Applications/Xcode12.3.app/Contents/Developer/usr/bin/git
```

即使PATH头部存在其它Toolchain目录，`xcrun --find xcodebuild`依然能解析到当前Toolchain的路径。

```
$ PATH="/Applications/Xcode12.3.app/Contents/Developer/usr/bin:${PATH}"

$ which xcodebuild                                                     
/Applications/Xcode12.3.app/Contents/Developer/usr/bin/xcodebuild

$ xcrun --find xcodebuild
/Applications/Xcode.app/Contents/Developer/usr/bin/xcodebuild
```

## 小结
1. 我们可以通过`xcode-select`方便地切换系统Toolchain；用`DEVELOPER_DIR`环境变量切换当前Shell session的Toolchain。
2. 在编写编译脚本时，有两种编写方法
    1. 使用桩命令（如`/usr/bin/xcodebuild`）运行当前Toolchain中的命令。
    2. 使用`xcrun`运行当前Toolchain中的命令，或`xcrun --find`在当前Toolchain中查找命令。

`xcrun`的工作细节，可以参考[这篇文章][3].

## Reference
[how-to-utilize-a-different-xcode-version-for-build-process-on-mac][1]

[multiple-xcode-versions-or-why-xcrun-is-your-friend][2]

[xcrun details][3]

[1]: https://support.macincloud.com/support/solutions/articles/8000042681-how-to-utilize-a-different-xcode-version-for-build-process-on-mac

[2]: https://dive.medium.com/multiple-xcode-versions-or-why-xcrun-is-your-friend-ed3935b054b

[3]: https://lapcatsoftware.com/articles/xcrun.html
