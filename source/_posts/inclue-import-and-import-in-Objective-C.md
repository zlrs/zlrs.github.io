---
title: '#inclue, #import, and @import in Objective-C'
date: 2020-12-29 20:44:35
tags: iOS Objective-C WWDC
---

> 最近在看WWDC中关于Objective-C的一些视频。正好看到[WWDC2013-404 Advances in Objective-C](https://developer.apple.com/videos/play/wwdc2013/404/)中提到Objective-C modules的内容。正好又联系上最近C++群里讨论比较多的 `C++20 modules`，感觉可以做个总结~

| 语法(semantic) | 归属模块                     | 描述                  |
|--------------|--------------------------|---------------------|
| #include     | 传统预处理器           | 文本插入(textual inclusion)，没有别的trick      |
| #import      | Objective-C的预处理器 | 文本插入。但能保证不重复引入头文件   |
| @import      | Objective-C(iOS 7.0+)    | Objective-C modules |

## #include
`#include`是一个支撑了C系语言几十年的机制。它就是简单的文本插入，没有别的trick。在C语言里，函数声明写在`.h`头文件中，函数实现写在`.c`源文件中。`#include "xxx.h"`就是先把`xxx.h`的文本插入到`#include "xxx.h"`所在的位置；如果插入的文本中还有`#include`就继续插入。
它有3个缺点，当然对应的也有workaround。
### 缺点1：可能重复引入头文件
```c
// add.h
inline int add(int a, int b) { return a + b; };
```

```c
// main.cpp
#include <add.h>
#include <add.h>  // compile error: redefinition of ‘int add(int, int)’

int main() {
    return 0;
}
```

#### 规避方法(workaround)
Header guard.
```c
// add.h
// Header Guard有项目名前缀+文件名前缀，是为了避免一不小心把源代码给替换了。
#ifndef PROJECTNAME_ADD_HEADER_GUARD
#define PROJECTNAME_ADD_HEADER_GUARD

inline int add(int a, int b) { return a + b; };

#endif
```

### 缺点2：宏替换问题
```c
#define FILE "data.txt"  // 本来我只想在本文件中使用这个宏
#include <stdio.h>  // 但是没想到 stdio.h 里的 FILE 也被替换成 "data.txt" 了
// 编译器会报一些很难理解的错误。
/* 
/usr/include/stdio.h:195:34: error: expected constructor, destructor, or type conversion before ‘;’ token
 extern FILE *tmpfile (void) __wur;
*/

int main() {
    // 读取 FILE 文件
    // ...
    return 0;
}
```

图：CI 对 Header Guard 过短给出的建议
![-w850](https://karl1b.blob.core.windows.net/mweb//2021/07/27/16118301586367.jpg)

#### 规避方法(workaround)
每次使用宏时都加上前缀，如项目名等，并且全用大写加下划线（因为一般符号很少全是大写的）。比如`#define FILE "data.txt"`，替换成`#define PROJECTNAME_DATA_FILE "data.txt"`。

### 缺点3：影响编译速度
(1) 头文件是嵌套`#include`的。一行普通的`#include <stdio.h>`，完全展开后可能有几千几万行代码。由于代码量很大，所以编译很耗时。
(2) 如果第(1)点的编译耗时是一次性的，那还可以接受。但是工程中有这样的情况: 几乎每个编译单元（源文件）都要引入同一个头文件，比如 `UIKit.h`。编译单元之间对头文件的编译产物无法复用。假如工程中有M个源文件，N个头文件，则项目的编译最多可以达到 MxN 的复杂度。这样的项目是非拓展性的(unscaleable)。
#### 规避方法(workaround)
使用pch(precompiled header). pch解决了上面提到的第(2)个问题。程序员将一些共用的头文件写在`.pch`文件中。`.pch`中的头文件会被预先编译好，并在每个源文件中默认引入。`pch`的确提高了编译速度。但是`pch`由于会被默认引入到每个源文件，这个解法其实很不好，因为它带来了名字空间污染(namespace polution)。
## #import 
`#import`是Objective-C预处理器引入的一个特性。它仍然是文本插入，但是可以保证同一个编译单元中，同一头文件的内容只被插入一次。
更详细的可以看StackOverflow: [What is the difference between #import and #include in Objective-C?](https://stackoverflow.com/questions/439662/what-is-the-difference-between-import-and-include-in-objective-c)。
## @import
我们可以先把目光从C系语言转移到JS、Python等语言。JS和Python都没有头文件，他们遵循的是modules的概念。一般`module`是一个文件，这个文件`export`出一些接口；其它文件可以`import`一个module，从而调用它暴露出来的接口。
在python中，我们可以import一个module任意次，而且当Python VM启动之后，只有第一次引入时才会执行module文件，然后把该module缓存到一个哈希表中，随后的引用将直接从哈希表中取得module。JS的`require(module)`也是类似的原理。
![python中可以import一个module任意次](https://karl1b.blob.core.windows.net/mweb//2021/01/23/16091712656983.jpg)

在iOS7(WWDC 2013)中，苹果为OC引入了modules机制，对应的关键字就是`@import`。
OC的modules有以下特点：

1. 不是文本插入(textual inclusion). 
2. 第二次被`@import`的module不需要重新编译。大大提高了编译速度。（尤其对于那些不使用或不经常维护`pch`的项目）
3. 通过`modulemap`文件关联umbrella头文件和module，并export submodule。顺便我们还可以在`modulemap`中写link指令，这样就可以自动链接上framework.
![-w849](https://karl1b.blob.core.windows.net/mweb//2021/01/23/16091729424173.jpg)
由于modules的Auto-Link特性，我们的工程在`Build Phases`中不需要引入`UIKit.framwork`, `Foundation.framwork`等依赖。
![-w856](https://karl1b.blob.core.windows.net/mweb//2021/01/23/16091731649362.jpg)

1. header is the truth。编译器最后做了什么，最后还是看header。类的接口也是依然写在header中，而不是`export`出来。
2. Xcode 新项目默认开启modules。现有项目(iOS 7.0+)一般都开启了这个功能。开启后，若该头文件能通过modulemap对应到某个module，则这条`#import`会被视做`@import`。因此开发者在OC工程中写的对系统库头文件和Cocoapods组件头文件的`#import`(如`#import <MapKit/MapKit.h>`)，大多数情况下已经不是直接的文本插入了，而是被Xcode视作`@import`(`@import MapKit`）。
![现有项目一般都开启了modules](https://karl1b.blob.core.windows.net/mweb//2021/01/23/16091727588022.jpg)

1. 我们自己的SDK也可以使用module的特性。看起来cocoapods会帮助我们做这件事。
![cocoapods自动生成的modulemap文件](https://karl1b.blob.core.windows.net/mweb//2021/01/23/16091734906367.jpg)

更详细的可以看[WWDC2013-404 Advances in Objective-C](https://developer.apple.com/videos/play/wwdc2013/404/)和 StackOverflow: [@import vs #import - iOS 7](https://stackoverflow.com/questions/18947516/import-vs-import-ios-7)。

## modules in C++
苹果的工程师将OC的这套modules实现借鉴到C++上，并做了在2012年做了一个分享。[C++ Modules proposal](https://www.youtube.com/watch?v=4Xo9iH5VLQ0)(2012年12月5日)

现在modules已经进入C++20，标准规定和苹果当初提出的方案挺相似的。也是基于[modulemap](http://clang.llvm.org/docs/Modules.html#module-map-language)实现的。


有关C++20 modules的更多信息，可以看下面的视频和资料。
[Demo: C++20 Modules](https://www.youtube.com/watch?v=6SKIUeRaLZE)

[C++20: An (Almost) Complete Overview - Marc Gregoire - CppCon 2020](https://www.youtube.com/watch?v=FRkJCvHWdwQ&t=232s)

[Modules in Clang 11](https://mariusbancila.ro/blog/2020/05/15/modules-in-clang-11/)

[How do I use C++ modules in Clang?](https://stackoverflow.com/questions/33307657/how-do-i-use-c-modules-in-clang)
