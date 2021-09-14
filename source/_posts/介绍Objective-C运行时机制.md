---
title: 介绍Objective-C运行时机制
date: 2021-09-14 15:53:39
tags: Objective-C runtime
---

> 本文写于2020年5月


> 本文是去年（2020年）阅读完《Object-oriented programming : an evolutionary approach》后整理的Objective-C runtime 的一些知识。
> 介绍了OC和SmallTalk-80语言的历史、早期的OC实现、OC中与运行时交互的3种方式、类的实现、消息绑定机制、super关键字与objc_msgSuper、KVO等内容。

Runtime源码：
https://opensource.apple.com/source/objc4/
https://github.com/0xxd0/objc4

## 历史
[Objective-C的历史](2021/09/14/Objetive-C的历史/)
[SmallTalk-80的历史](2021/09/14/SmallTalk-80的历史/)

## 设计哲学
- 尽可能延迟决策，动态绑定。
- 面向对象特性完全由 Runtime 提供
- objc对象没有方法调用，而是消息发送(message expressions)。消息由Runtime动态解析到函数地址
- 对象都分配在堆上。Runtime内的内存管理模块做“引用计数”
  - 注意tagged pointer表面上是对象，实际上不是对象
- 支持在运行时获得对象的信息（反射；自省）
- 支持在运行时添加新类、新方法
- 如果对象处理不了消息，还可以做消息转发
## 早期（1980s）的OC实现
早期的OC就是一个预处理器地位的存在。将面向对象语法翻译为C语言，然后再交给C编译器去编译。
![](https://cdn.zlrs.site/mweb/2021/09/14/16316113089181.jpg)

## runtime 版本
Runtime 的实体是一个动态链接库(libobjc.dylib)。Objective-C语言很大程度上就是这个动态链接库。这个库主要有两个大版本：
- modern runtime: ObjC 2.0 for iPhone apps and 64bit apps on OSX.
- legacy runtime: ObjC 1.0 for 32bit apps on OSX

## 和Runtime交互的形式
1. 通过Objective-C Source Code
编译器将源代码中的类和方法转换成数据结构和动态特性的函数。数据结构从源码中提取类定义、分类定义、协议声明（class and category definitions and in protocol declarations）的信息。运行时系统的主要功能就是消息发送，由源码中的消息发送语句触发（invoked by source-code message expressions）.
2. 通过NSObject的一些方法
NSObject的一些方法会问询runtime获得信息，一般为类方法。这些方法也被称为自省方法（self-introspect）
- isKindOfClass:
- isMemberOfClass:
- respondsToSelector:
- conformsToProtocol:
- methodForSelector: （provides the address of a method’s implementation）
3. 直接调用运行时库函数 `#import <objc/runtime.h>`
运行时库函数是C语言接口。其API可分为两种类型。
- 第一种类型允许你使用C函数去使用部分编译器的能力（allow you to use plain C to replicate what the compiler does when you write Objective-C code）. 
- 第二种类型 form the basis for functionality exported through the methods of the NSObject class.

## 类的实现
源码：https://opensource.apple.com/source/objc4/objc4-235/runtime/objc-class.h.auto.html
```cpp
struct objc_class {                        
    struct objc_class *isa;  // 一般指向metaclass
    struct objc_class *super_class;  // 指向父类
    const char *name;  // 类名
    long version;      // 类的版本
    long info;
    long instance_size;  // 实例变量大小之和
    struct objc_ivar_list *ivars;   // 保存实例变量的名字、类型、offset等信息，注意类型

    struct objc_method_list **methodLists;  // 函数的查询表，注意类型

    struct objc_cache *cache;  // 消息查询结果的缓存表
    struct objc_protocol_list *protocols;
};
```
### 关于isa
指向metaclass。
### 关于super_class
指向父类。
### 关于version
用来标识类的接口的变动，也就是为类的接口打版本。在序列化时特别有用，因为可以用它来表明类的实例变量的布局信息是否发生了变化。NSObject有类方法：setVersion:
关于instanceSize
alloc函数怎么知道要为实例申请多大的空间？就是看类对象的instanceSize。下面是new操作的一部分代码，节选自NSObject.mm中的static ALWAYS_INLINE id _class_createInstanceFromZone(Class cls, 后面参数省略...)函数

```Objective-C
size = cls->instanceSize(extraBytes);
if (outAllocatedSize) *outAllocatedSize = size;

id obj;
if (zone) {
    obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
} else {
    obj = (id)calloc(1, size);
}
```
### new方法的早期实现
节选自《Object-oriented programming : an evolutionary approach》
![](https://cdn.zlrs.site/mweb/2021/09/14/16316112931084.jpg)

### 关于methodLists
1. 注意：是methodLists而不是methodsList
2. methodLists是多个method List的线性表，其中最后一个元素是base method list，其它元素都是category method list
3. 我们写类文件的时候，会写类方法和实例方法，实际上它们在编译后会注册到2个Class结构体中，其中实例方法进入类的Class结构体，类方法进入类的ISA，也就是metaclass，的Class结构体。具体请看方法的实现和消息机制。
4. 可以在method list中查找SEL的IMP。这个列表可以是有序的，那么可以用二分查找；如果是无序的，可以用线性查找。

### 关于cache
消息查找的缓存。大多数消息查找的耗时很少，因为它们会直接命中缓存。
class相关runtime API
1. isKindOfClass:
判断对象是否是该类或该类派生类的实例。入参先去和ISA比较，再一路沿着super_class去比较。

早期Object.m源码
```Objective-C
- (BOOL)isKindOfClassNamed:(const char *)aClassName
{
    register Class cls;
    for (cls = isa; cls; cls = ((struct objc_class *)cls)->super_class) 
        if (strcmp(aClassName, ((struct objc_class *)cls)->name) == 0)
            return YES;
    return NO;
}
```
近期NSObject.mm源码
```Objective-C
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = self->ISA(); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```
2. isMemberOfClass:
判断对象是否是该类的直接实例。其实就是看receiver的isa和入参是不是一样。

早期Object.m源码
```
- (BOOL)isMemberOfClassNamed:(const char *)aClassName {
    return strcmp(aClassName, ((struct objc_class *)isa)->name) == 0;
}
```
近期NSObject.mm源码
```
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}
- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```
注：以上的早期源码和近期的逻辑是一样的。可以看一下近期源码中class方法的实现，其实也是返回ISA。
```
// NSObject.mm
- (Class)class {
    return object_getClass(self);
}

// objc-class.mm
/***********************************************************************
* object_getClass.
* Locking: None. If you add locking, tell gdb (rdar://7516456).
**********************************************************************/
Class object_getClass(id obj) {
    if (obj) return obj->getIsa();
    else return Nil;
}
```

## 方法的实现
```
源码：https://opensource.apple.com/source/objc4/objc4-235/runtime/objc-class.h.auto.html
typedef struct objc_method *Method;

struct objc_method {
        SEL method_name;
        char *method_types;
        IMP method_imp;
};

typedef struct objc_method *Method;

struct objc_method_list {
        struct objc_method_list *obsolete;

        int method_count;
#ifdef __alpha__
        int space;
#endif
        struct objc_method method_list[1];        /* variable length structure */
};
```

编译后，OC方法的数据结构是一个 `struct；`方法体会变为一个C函数，这个C函数前两个参数为对象指针/类对象指针和当前方法的SEL，参数列表为`(id self, SEL _cmd, ...)`
### IMP的前2个参数
1. The receiving object
对应我们在OC源代码的类方法或者实例方法中写的`self`。实例方法的`self`为该对象的指针。类方法的`self`为该类对象的指针。
2. The selector for the method
对应OC源码中的`_cmd`。
### 方法的种类
C++中，方法被称为成员函数与静态成员函数。
Java中，方法被称为类方法与实例方法。
OC中，方法可以被分为：
1. 实例方法（instance method）。方法前面是减号。这些方法会注册到本类的方法表中。
2. 工厂方法（factory method）。“工厂”一词不是指设计模式，而是指类的工厂类（也就是类的metaclass）的方法。方法前面是加号。这些方法会注册到本类的工厂类，也就是metaclass的方法表中。
这里提到的方法表就是 objc_class.methodLists

## 对象内存模型与消息派发
### 概述
1. 对象只存储状态（实例变量）；类只存储行为（方法）。
  1. 类和对象都有isa指针和superclass指针，作为其状态的一部分，即实例变量。
2. 类和对象是一种模板-实例关系。且类和对象相对的概念；对类A的meta class而言，类A也是对象；对NSObject meta class而言，NSString meta class也是对象。NSObject比较特殊，它就是root的类，它不是任何人的对象（superclass指向nil）。
3. 类方法和实例方法的派发，其实都是一样的。都是先找到receiver的isa，然后再一直向上找super class，直到到达NSObject为止。
4. 因此所有的方法，包括类方法和实例方法，最终都是派发到NSObject。
5. 对象的isa指针指向其类对象。类对象的isa指针指向类对象的metaclass对象；super class指针指向超类。metaclass对象的isa指针指向NSObject的metaclas对象。
### 对象图
![](https://cdn.zlrs.site/mweb/2021/09/14/16316113683323.jpg)


### 对象图（早期实现）
![](https://cdn.zlrs.site/mweb/2021/09/14/16316113769760.jpg)


下面这张图节选自《Object-oriented programming : an evolutionary approach》，其中 Pen Software IC 指的就是Pen类。

### 实例的消息派发
![](https://cdn.zlrs.site/mweb/2021/09/14/16316113875747.jpg)


### 类对象的消息派发
![](https://cdn.zlrs.site/mweb/2021/09/14/16316114039154.jpg)


## 动态绑定与消息机制
这是OC与C最大的不同。C的函数调用，在编译期完成函数绑定，函数地址会直接体现在每次函数调用中。而OC对象是通过消息机制在运行时完成绑定。
### 概览
Here is roughly how objc_msgSend function works:
1. If the receiving object is nil, the message is redirected to nil receiver if any. The default behavior is to do nothing.
2. Check the class’s cache. If the implementation is already cached, call it.
3. Compare the selector to the selectors defined in the class. If a match is found, call the matched implementation. Otherwise, check its superclass until there is no superclass.
4. Call +resolveInstanceMethod:/+resolveClassMethod:. If it returns YES, it means the selector will resolve this time. So go to step 2 and start over. It is the place that you call class_addMethod to dynamically provide an implementation for the given selector.
5. Call -forwardingTargetForSelector:. If it returns non-nil, send the message to the returned object instead. Note that return self here will result in an infinite loop.
6. Call - methodSignatureForSelector:. If it returns non-nil, create an instance of NSInvocation, and pass it to -forwardInvocation:.
7. The implementation of the given selector cannot be found. It will call -doesNotRecognizeSelector: on the receiving object. The default implementation throws an exception.

### objc_msgSend
在OC中，消息是在运行时绑定的。编译器做的事情是：
1. 将`[receiver message]`消息发送式转换为`objc_msgSend(receiver, selector, arg1, arg2, ...)`函数。
2. 为每个类和对象建立数据结构，这个数据结构中包括 isa 指针和 class dispatch table（方法名到方法实现地址的查询表）。

`objc_msgSend`是一种messager。messager有几种，比如在向super发送消息时，会生成类似`objc_msgSuper`这样的messager。
这也是super关键字的原理。super并不是一个变量（self是），而是生成一种新的messager——msgSuper。
objc_msgSend做的事：
1. 根据receiver，查找SEL对应的IMP地址。这个查找分成3个层级
  1. 最高层，是那些需要被查找的类。这些类是继承树的一条路径，从receiver->isa开始，一直往继承树的根部，也就是NSObject的这样一条路径。objc_msgSend会按照顺序被查询路径上的每个类。
  2. 中间层，是对单个类的查询。每个类可能有多个函数列表（method list）。举个例子，对于NSObject来说，它首先有一个类文件中直接实现的method list，也会有很多category method list，这些是通过category添加上去的。
  3. 最底层，是对单个method list的查询。method list中存放了SEL到IMP的映射。它可能是有序的，也可能是无序的；查询的方式有二分查找和线性查找。
2. 调用函数指针，传入2个隐藏的默认参数：receiver 和 selector，以及消息的参数。
3. 返回函数调用的返回值。

#### 动态方法解析 Dynamic Method Resolution
```
+ (BOOL) resolveInstanceMethod:
+ (BOOL) resolveClassMethod:
```
动态方法解析是指为类添加方法（Method）到函数地址（IMP）的映射。
一个OC方法不过是一个至少接收2个参数——self和_cmd的C函数而已。使用class_addMethod函数，你就能动态地为一个类添加一个方法和方法实现。
resolveInstanceMethod:在OC的消息转发机制（forwarding mechanism）之前被调用，提供动态为一个类添加方法的机会。If respondsToSelector: or instancesRespondToSelector: is invoked, the dynamic method resolver is given the opportunity to provide an IMP for the given selector first.
```Objective-C
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}

+ (BOOL) resolveInstanceMethod:(SEL)aSEL {
    if (aSEL == @selector(resolveThisMethodDynamically)) {
        class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSel];
}
```
#### 转发 Forwarding
如果对象认为自己不能响应消息，可以将消息转发给另一个对象。
若resolveInstanceMethod:返回NO，则对象可转发该消息。这其中又分为2小步。
##### 1.  - forwardingTargetForSelector:
请接受者看看是否有其它对象（replacement receiver）能请求该消息。若有，运行时系统会把消息转给那个对象，消息转发结束。`- (id)forwardingTargetForSelector:(SEL)aSelector;`. 若可以处理，则返回处理消息的对象；若不能处理调用返回nil或者`[super forwardingTargetForSelector:]`。请注意，在这步中无法修改所转发的SEL。但是这一步的好处是代价比较低。
##### 2. - methodSignatureForSelector:  / - forwardInvocation:
若上一步函数返回nil，则启动完整的消息转发机制。先调用`- methodSignatureForSelector: `，若返回不为nil，则根据返回值将当前消息封装到NSInvocation对象中，调用`forwardInvocation: `，再给receiver最后一次机会处理该消息。这一步可以为消息追加参数或者修改SEL。
实现此方法时，若某消息不应由本类处理，则可以调用超类的同名方法。这样的话，继承树中的每个类都有机会处理此消息。优点是灵活，缺点是和相比`forwardingTargetForSelector:`代价较高。
doesNotRecognizeSelector
默认的实现是抛出异常

## 异常处理
如果消息经过整套流转过程后，还不能被处理，就会产生异常。我们可以用try-catch捕获异常，也可以设置全局异常处理函数。
### 受保护的代码
```
NSObject* obj = [[NSObject alloc] init];
 
@try {
  // Attempt send a message that the receiver cannot understand
  [obj performSelector:@selector(unknowMessage)];
}
@catch (NSRangeException *exception) {
  NSLog( @"Name: %@", exception.name);
  NSLog( @"Reason: %@", exception.reason);
}
@finally {
}
```
### 执行UncaughtExceptionHandler
设置Handler：NSSetUncaughtExceptionHandler
Sets the top-level error-handling function where you can perform last-minute logging before the program terminates. The program then terminates, regardless of the actions taken by the uncaught exception handler.
Terminating app due to uncaught exception 'NSInvalidArgumentException'
异常最终会导致crash。

## 思考
### 方法是消息的handler
方法定义在哪里并不重要，重要的是第一个参数self是谁。
类更像是一个存放方法的容器，只不过这些方法一般是围绕实例的私有变量工作的，可以产生内聚。
A类的方法和B类的方法本质上没有什么不同，就是一个接受self和SEL作为前两个参数的C函数而已。假设A类和B类不在一棵继承树中，如果硬要拿B类的IMP来处理A类消息，应该也可以，只是一般不会这样做。
如果A类和B类有继承关系，那就很自然了。例如，不管给哪个类发new消息，一般都是解析到NSObject定义的new方法的IMP来处理。

### 类函数和实例函数是相对的
本类的类函数是本类的模板类的实例函数，加号和减号只是告诉runtime该方法应该注册到本类还是本类的模板类的函数表中。

### isa与superclass的语义
isa 描述实例-模板的关系；superclass描述子类-父类的继承关系。

### 类消息和实例消息的派发方法是相同的
类消息和实例消息的派发，都是基于相同的算法。即先查询receiver的isa的函数表，再查询receiver的isa的superclass的函数表，再一直沿着被查询对象的superclass，直到查询到NSObject。若此时还没有查询到SEL，就进入后面的过程，即调用resolveInstanceMethod:等等。

## Further Topics
### SmallTalk-80的实现
《Smalltalk-80: The Language and its Implementation》, Addison-Wesley, 1983
### 关联对象
#### 使用
https://stackoverflow.com/questions/8733104/objective-c-property-instance-variable-in-category
给Runtime提供的键必须是全局唯一的，比如一个静态变量的指针或者SEL。
#### 实现
直接看代码。https://juejin.im/post/6844903796825391117
#### Runtime APIs
- objc_setAssociatedObject
- objc_getAssociatedObject
- objc_removeAssociatedObjects

### KVO
观察者模式
原理：通过isa Swizzling添加中间类；中间类重写setter，给观察者发送属性变更通知、重写class方法返回原本的父类以隐藏自己。
![](https://cdn.zlrs.site/mweb/2021/09/14/16316115725874.jpg)

### Method Swizzling
#### Getting a Method's Address
NSObject 的方法, methodForSelector: 传入一个selector，返回一个函数指针。
```
void (*setter)(id, SEL, BOOL);
int i;
 
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

#### Dynamic loading
An Objective-C program can load and link new classes and categories while it’s running. The new code is incorporated into the program and treated identically to classes and categories loaded at the start.
NSBundle类为Dynamic loading 提供了接口。
在运行时向类中添加方法和方法实现（通过`class_addMethod(Class, SEL, IMP, ...)`函数），本质上是在类的方法地址查询表中添加记录。
Method Swizzling 是在运行时向类中交换两个方法的实现。本质上是交换类的方法地址查询表中的两个键的值。
开发者常用此功能向原有实现中添加新功能，如UI自动数据上报。
#### Related APIs
1. Method class_getInstanceMethod(Class class, SEL selector)
2. void method_exchangeImplementations(Method m1, Method m2);
[Apple Doc](https://developer.apple.com/documentation/objectivec/1418769-method_exchangeimplementations?language=objc)
3. IMP method_getImplementation(Method m);
4. IMP method_setImplementation(Method m, IMP imp);
Example:
```
NSString *str = @"lowercase-UPPERCASE";

Method lowercaseMethod = class_getInstanceMethod([NSString class], @selector(lowercaseString));
Method uppercaseMethod = class_getInstanceMethod([NSString class], @selector(uppercaseString));
method_exchangeImplementations(lowercaseMethod, uppercaseMethod);

NSLog(@"call lowercaseString: %@", [str lowercaseString]);
NSLog(@"call uppercaseString: %@", [str uppercaseString]);
```

### OC的对象模型
- 类 = 私有数据 + 共享操作
- 数据的“链接”
  - 对象布局
- 操作的“链接”
  - 动态派发机制

### Super
#### 问题
![](https://cdn.zlrs.site/mweb/2021/09/14/16316116657873.jpg)

#### 分析
分析下面这段代码。
```
#import <Foundation/Foundation.h>

@implementation TestSuperMessager : NSObject

- (void)sayHello {
    [self someMethod];
    [super someMethod2];
}

@end
```
将其重写为C++: `clang -rewrite-objc main.m -o testMsg.cpp`
```
// @implementation TestSuperMessager : NSObject

static void _I_TestSuperMessager_sayHello(TestSuperMessager * self, SEL _cmd) {
    ((id (*)(id, SEL, …))(void *)objc_msgSend)   // objc_msgSend函数，做了类型cast
    ((id)self, sel_registerName(“someMethod”));  // 实参

    ((id (*)(__rw_objc_super *, SEL, ...))(void *)objc_msgSendSuper) //函数
    ((__rw_objc_super){
       (id)self, 
       (id)class_getSuperclass(objc_getClass(“TestSuperMessager"))
    }, sel_registerName(“someMethod2”));  // 实参
}
// @end

// 以下是Runtime的定义，和上面结合起来看
OBJC_EXPORT id _Nullable
objc_msgSendSuper(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)

/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;
    /// Specifies the particular superclass of the instance to message. 
    __unsafe_unretained _Nonnull Class super_class;
   /* super_class is the first class to search */
};
```
#### 小结
- 编译器会为super关键字生成objc_msgSendSuper的messager
- 向super关键字发送消息，receiver还是self。只不过方法的查询起点由本类变成了父类。
- 由于语法的原因，我们可能会误解super是消息的receiver，其实并不是。
- super的作用是指定方法搜索起点为父类，消息的receiver仍然是self。
- [self class]和[super class]找到的IMP都是NSObject中的那个IMP。而找到IMP后，IMP的第一个实参还是self。所以调用的函数和传递的参数都相同，最后输出的类名当然也是相同的。

## metaclass从何处来，是怎么被集成进应用的？
和编译、链接和加载有关。TODO

## Reference
1. 什么是运行时 https://www.zhihu.com/question/27179396
2. OC消息派发和流转机制
  1. 概述 https://medium.com/@guanshanliu/how-message-passing-works-in-objective-c-9e3d3dd70593
  2. 消息分发机制 https://stackoverflow.com/questions/982116/objective-c-message-dispatch-mechanism
  3. https://juejin.im/post/5d47cfcb5188252d33698994#heading-9
3. 从指令层面去分析objc_msgsend() http://www.friday.com/bbum/2009/12/18/objc_msgsend-part-1-the-road-map/
4. OC对象内存模型 
  1. 对象模型 https://devalot.com/articles/2011/11/objc-object-model.html
  2. https://zhuanlan.zhihu.com/p/34270138
  3. https://developer.apple.com/library/archive/documentation/General/Conceptual/CocoaEncyclopedia/ObjectModeling/ObjectModeling.html
  4. https://devalot.com/articles/2011/11/objc-object-model.html
5. SmallTalk相关
  1. Reflective Facilities in Smalltalk-80 http://www.laputan.org/ref89/ref89.html
  2. https://techbeacon.com/app-dev-testing/how-learning-smalltalk-can-make-you-better-developer
6. 《Object-oriented programming : an evolutionary approach》，出版于1986年，讲述了OOP的早期理论和各种实现方法，以及作者为什么要把OC设计成这样。作者是OC的发明者Cox, Brad https://archive.org/details/objectorientedpr00coxb/page/n17/mode/1up
