---
title: 探索Xcode命令行工具系统（二）：git命令的调用过程
date: 2021-08-06 14:39:55
tags: macOS iOS
---

上一节我们介绍了Xcode和Command Line Tools。`git`是Command Line Tools的一部分（更准确地说，是其中BSD Tools的一部分）。这一节我们通过`git`命令来追踪Command Line Tools是怎么调用的。

直接查看git命令的位置，可以发现是位于`/usr/bin/git`. 
```fish
zlrs@C02D ~/B/P/b/t/tulsi (master)> which git 
/usr/bin/git
zlrs@C02D ~/B/P/b/t/tulsi (master)> ls -l (which git)
-rwxr-xr-x  1 root  wheel  137616  1  1  2020 /usr/bin/git
```

但是，在未安装Command Line Tools的情况下，虽然`/usr/bin/git`存在，但是依然无法使用git的。如果调用git，将提示未安装Command Line Tools并exit。

实际上`/usr/bin/git`并非真实的git程序。`/usr/bin/git`只是一个桩，实际上是调用xcrun去执行当前Xcode Developer directory中的git（如`/Applications/Xcode.app/Contents/Developer/usr/bin/git`）. 

做一个简单的实验，通过DEVELOPER_DIR环境变量指定Xcode Developer directory之后，可以发现xcrun无法找到真正的git命令的路径。
![-w734](https://cdn.zlrs.site/mweb/2021/08/06/16282457725258.jpg)

这一点也可以通过追踪`/usr/bin/git`的系统调用来验证。追踪前需要先移除桩二进制的签名。
```
mv /usr/bin/git ./
sudo codesign --remove-signature ./git
sudo dtruss ./git -h
```

![-w1625](https://cdn.zlrs.site/mweb/2021/08/06/16282465341716.jpg)

![-w1481](https://cdn.zlrs.site/mweb/2021/08/06/16282465142731.jpg)


完整代码见[gist](https://gist.github.com/zlrs/4aa3f284eb0cb23fb2e1da0b16945814). 

<script src="https://gist.github.com/zlrs/4aa3f284eb0cb23fb2e1da0b16945814.js"></script>
