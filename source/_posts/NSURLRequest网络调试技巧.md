---
title: NSURLRequest网络调试技巧
date: 2021-02-07
tags: 网络 iOS macOS
---

## 快速阅读
1. 本文提供了一个`NSURLRequest`分类，便于在运行时捕获URL请求对象的请求信息。
2. 本文介绍了VSCode插件`REST Client`的使用方法。
3. 本文介绍了如何将上面两者配合使用，从而方便iOS/macOS客户端同学调试网络请求。

这个方法原理是先把NSURLRequest以「HTTP格式」或「cURL格式」打印出来，然后利用VSCode的「REST Client」插件或者终端随时修改请求内容，便于调试请求。

## 调试准备

1.  在客户端代码中添加下面的分类

```
@implementation NSURLRequest (DEBUG)

#ifdef DEBUG

/* 生成VSCode「REST Client」插件所需要的请求格式（HTTP 语言） */

- (NSString *)debug_VSCodeRESTClientPlugin_HTTP {

    NSMutableString *result = [[NSMutableString alloc] init];

    [result appendString:[NSString stringWithFormat:@"%@ %@ %@\n", self.HTTPMethod, self.URL.absoluteString, @"HTTP/1.1"]];

    

    NSDictionary *fields = self.allHTTPHeaderFields;

    for (NSString *headerName in fields) {

        NSString *headerVal = fields[headerName];

        [result appendString:[NSString stringWithFormat:@"%@: %@\n", headerName, headerVal]];

    }

        

    NSString *body = [[NSString alloc] initWithData:self.HTTPBody encoding:NSUTF8StringEncoding];

    [result appendString:[NSString stringWithFormat:@"\n%@\n\n###", body]];

    

    return result;

}



/* 生成VSCode「REST Client」插件所需要的请求格式（cURL 格式） */

- (NSString *)debug_VSCodeRESTClientPlugin_cURL {

    NSMutableString *result = [[NSMutableString alloc] init];

    [result appendString:[NSString stringWithFormat:@"curl -i -X %@ ", self.HTTPMethod]];

    [result appendString:[NSString stringWithFormat:@"\"%@\" ", self.URL.absoluteString]];

    

    NSDictionary *fields = self.allHTTPHeaderFields;

    for (NSString *headerName in fields) {

        NSString *headerVal = fields[headerName];

        [result appendString:[NSString stringWithFormat:@"-H \"%@: %@\" ", headerName, headerVal]];

    }

        

    NSString *body = [[NSString alloc] initWithData:self.HTTPBody encoding:NSUTF8StringEncoding];

    [result appendString:[NSString stringWithFormat:@"-d \"%@\"", body]];

    

    return result;

}

#endif

@end
```

2.  在VSCode上安装 **REST Client** 插件
**REST Client** 是一个用于调试网络请求的VSCode插件。其功能和Postman类似。其特点是纯文本操作，不需要用鼠标在GUI上点点点，熟练使用后调试效率较高。

**REST Client** 的使用方式非常简单，在VSCode中新建一个`.http`或`.rest`拓展名的文件，然后在文件中书写「HTTP格式」或「cURL格式」的请求代码。点击代码头部或右键菜单中的`send Request`发送请求，就可以在右侧看到响应了。

![](https://raw.githubusercontent.com/Huachao/vscode-restclient/master/images/response.gif)


## 使用方法（HTTP 格式）

1.  在需要调试的地方调用分类方法，把URL请求对象“捕获”下来

```
NSMutableURLRequest *request = xxx;

NSLog(@"%@", [request debug_VSCodeRESTClientPlugin_HTTP]);
```

2.  在VSCode中打开一个http后缀名的文件，把输出的内容贴到xxx.http文件里。
![](https://cdn.zlrs.site/mweb/2021/07/28/16274650141059.jpg)

3.  在右键菜单中点击 sendRequest，就可以发出请求，并在右侧看到服务端的响应了。
![](https://cdn.zlrs.site/mweb/2021/07/28/16274650346776.jpg)

4.  如果请求参数需要调整，直接在文件中修改并重新发送请求即可。比如上面的截图中，服务端查询返回的是一个空数组。这时候我们就可以方便地修改请求query，很快就能查看修改后的效果。

## 使用方法（cURL格式）

1.  打印cURL格式命令

```
NSMutableURLRequest *request = xxx;

NSLog(@"%@", [request debug_VSCodeRESTClientPlugin_cURL]);
```

2.  可以复制到终端中使用，也可以在VSCode「REST Client」插件中调用。
通过终端使用:
![](https://cdn.zlrs.site/mweb/2021/07/28/16274650482513.jpg)
通过「REST Client」插件使用：
![](https://cdn.zlrs.site/mweb/2021/07/28/16274650604452.jpg)

## 其他

目前只考虑到了HTTPBody为文本的情况，如果HTTPBody是二进制就需要自己修改下分类代码。
