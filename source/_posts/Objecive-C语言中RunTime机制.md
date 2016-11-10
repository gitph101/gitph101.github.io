---
title: Objecive-C语言中RunTime机制
date: 2016-11-10 14:43:38
tags:
- OC
- iOS
- runtime
categories: 
- iOS
---

为了充分介绍Objecive-C语言中RunTime机制，使读者对RunTime机制有一个清晰的了解。本文将从以下几个部分来介绍相关知识：

- 第一部分将介绍Objecive-C语言的基本特性。
- 第二部分将介绍Objecive-C类对象与isa指针的本质。
- 第三部分将介绍RunTime整个流程。
- 第四部分将介绍RunTime的常见应用场合。<!-- more -->

# 第一部 Objecive-C语言基本特性

Objecive-C(以下简称OC)是一种采用消息结构、面向运行时、动态的面向对象的语言。其运行时所执行的代码由运行环境来决定。它本质上是C的超集。**OC中重要的工作由“运行期组建”（runtime component）而非编译器完成。**

备注：一般来说面向对象的语言有两种实现形式：一种是消息结构，另一种是采用函数调用。它们两者的区别在于。后者运行时所执行的代码由编译器决定，也就是说采用函数调用的语言在编译期编译器就会查找出运行是代码所执行的方法。采用消息结构（Objecive-C）语言只有在运行时确定所执行的代码方法。

# 第二部 Objecive-C中对象与isa指针。

“对象”是面向对象语言的基本构造单元，开发者可以通过对象来存储和传递数据。理解“对象”对于理解消息传递机制有着重要的意义。

要认识什么是isa指针，我们得先明确一点：
**在OC中，任何类的定义都是对象。类和类的实例（对象）没有任何本质上的区别。任何对象都有isa指针。**

描述OC对象所用的数据类型定义在运行期程序文件的头文件里：

```
struct objc_object{
      Class isa  OBJC_ISA_AVAILABILITY;
}
typedef struct objc_object *id;
```

 由此可见每个对象的首个成员时Class类的变量，该对象称为isa指针。它指向对象的类。

类的定义也在头文件里：

``` objective-c
@interface NSObject <NSObject> {
  Class isa  OBJC_ISA_AVAILABILITY;
}
```
看出来NSObject定义了一个成员变量，我们继续寻找:
```objective-c
typedef struct objc_class *Class;
/// Represents an instance of a class.
```

**Class 是一个 objc_class 结构类型的指针, id是一个 objc_object 结构类型的指针。**

继续打开objc_class,我们找到class的定义：

```objective-c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                                        &OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
```

从上可以看出：isa本质上是一个Class 类型的指针。每个实例对象有个isa的指针,他指向对象的类。而Class里也有个isa的指针, 指向meteClass(元类)。元类保存了类方法的列表。当类方法被调用时，先会从元类查找类方法的实现，如果没有，元类会向他父类查找该方法。同时注意的是：元类（meteClass）也是类，它也是对象。元类也有isa指针,它的isa指针最终指向的是一个根元类(root meteClass).根元类的isa指针指向本身，这样形成了一个封闭的内循环。

下面这张图很好的解释了类的继承关系：

![449095-3e972ec16703c54d.png](http://upload-images.jianshu.io/upload_images/449095-b5ed6ceedfbc2b39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



- 每一个对象本质上都是一个类的实例。其中类定义了成员变量和成员方法的列表。对象通过对象的isa指针指向类。
- 每一个类本质上都是一个对象，类其实是元类（meteClass）的实例。元类定义了类方法的列表。类通过类的isa指针指向元类。
- 所有的元类最终继承一个根元类，根元类isa指针指向本身，形成一个封闭的内循环。



# 第三部分：RunTime机制

RunTime指一个程序在运行（或者在被执行）的环境。简称运行时。它指的是系统在运行的时候的一些机制，其中最主要的是消息机制。

OC中RunTime本质上是一套比较底层的纯C语言API, 属于1个C语言库, 包含了很多底层的C语言API。 在我们平时编写的OC代码中, 程序运行过程时, 其实最终都是转成了runtime的C语言代码。RunTime提供了一些使得对象之间能够传递消息的重要函数，并且包含了创建类实例所用的全部逻辑。

## 基础知识

在OC中，給某个对象传递消息可以这样

``` objective-c
 id returnValue ＝ ［obj messageName:parameter];
```

其中obj是接受对象,`messageName`是选择子`(selector),parameter`是参数。我们把`selecto`r和`parameter`结合起来称为消息。

编译器会把以上消息转化为C语言函数调用:

```objective-c
id returnValue ＝ objc_msgSend(id obj,SEL messageName,parameter).
```

`objc_msgSend`是消息传递机制中核心函数。其原型如下：

```objective-c
 void  objc_msgSend(id self,SEL cmd,...)
```

## 消息传递机制主要分为以下几个阶段：

### 第一阶段：消息传递机制（pass a message）

编译把OC中方法转化`objc_msgSend`形式后。`objc_msgSend`函数首先通过接受对象（obj）的isa指针找到接收对象(obj)对应的类`（class）`。在类`（Class）`中先去cache中 通过选择子`（SEL`）查找对应函数的方`method（）`，若 cache中未找到。再去方法列表`（methodList）`中查找，若方法列表`（methodList）`中未找到，则去`superClass`中查找。若能找到，则将method加 入到cache中，以方便下次查找，并通过method中的函数指针跳转到对应的函数中去执行。若没有找到，消息传递机制将进入第二阶段消息转发阶段（消息转发机制）。

**缓存**：每一个类都都有一块缓存（cache）。其存储的是“快速映射表”。objc_msgSend会将方法的匹配结果缓存在快速映射表中。

消息派发系统会将类的方法列表的名称映射到相关方法的实现上。以此来找到应该调用的方法。这些方法均以函数指针(IMP)的形式来表示。其原型是：

```objective-c
id（＊IMP）（id，SEL，...）
```

#### 方法调配技术：运行此特性可以改变相关方法的实现。

NSString 有可以响应以下几个方法：
![417684A6-0D92-4EE3-961C-B16FFE6D5ED8.png](http://upload-images.jianshu.io/upload_images/449095-9843bfca776f234e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
OC运行系统提供了几个方法能够操控这张表。开发者可以向其中新增选择子,也可以改变选择子的实现，还可以交换选择子所映射的指针。经过几次操作可以变成以下：
![0D1876D1-7098-42E2-B4B3-08AF769CE77F.png](http://upload-images.jianshu.io/upload_images/449095-840df668989dc4ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对比两张表，我们可以发现下面新增加了一个选择子。交换两个选择子的实现。可以看出方法调配技术的强大之处。

交换两个方法的实现，参数为方法的实现。

``` objective-c
void method_exchangeImplementations(Method m1,Method m2)
```

得到方法的实现：

```objective-c
Method class_getInstanceMethod(Class aClsaa,SEL aSelector)
```

实际应用中交换两个方法的实现意义不大。一般是在类别中新增加一个方法，然后交换两个方法的实现。以此来改变类的方法的实现。

![消息传递](http://upload-images.jianshu.io/upload_images/449095-fca5ec800efe1bb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第二阶段：消息转发机制（message forwarding）

消息转发机制分为两个小阶段

####  1.动态方法解析

第一阶段运行系统会先征询接受者所属的类看其是否动态添加方法以处理当前的这个未知选择子（SEL）。
在对象收到无法解读的消息后，便会调用所属类的下列方法：

```objective-c
＋（BOOL）resolveInstanceMethod:(SEL)selector
```

该方法参数就是需要处理的未知选择子。返回值是表示这个类是否能新增一个方法处理这个未知选择子。在处罚完整消息转发机制前本类有机会增加一个方法处理这个未知选择子。在此类方法中可以通过`class_addMethod` 增加一个方法处理未知选择子。

如果是类方法将会触发下面这个方法：

```objective-c
＋（BOOL）resolveClassMethod:(SEL)selector
```

#### 2.完整的消息转发机制：

如果第一阶段没有成功的处理未知选择子，那么接受者自己再也没有办法动态的添加方法来响应该未知选择子的消息了 。此时运行系统会启动完整的消息转发机制。这里又分为两个小阶段：

#####  第一阶段:

会请接受者看看有没有其他对象能处理这条消息。若有，运行系统会把消息转给那个对象（备援的接受者）,消息转发结束 。

```objective-c
-(id)forwardingTargetForSelector:(SEL)selector
```

若没有将进入第二阶段。

##### 第二阶段

运行系统会把消息的全部细节封装到NSInvocation对象中，再給接受者一次机会，另其解决当前未知选择子`（SEL`

如果消息转发到了这一个阶段，运行系统就会启用完整的消息转发机制。首先创建NSInvocation对象。把消息的全部细节封装于其中。此对象包括选择子，目标对象和参数。触发`NSInvocation`对象时，消息派发系统（message－dispathch－system）将亲自把消息指派給目标对象。

此步骤运行系统会调用下面方法来处理:

```objective-c
－(void)forwardingInvocation:(NSInvocation)invocation。
```

这个方法可以有多种实现形式。可以很简单，改变消息的调用目标，使消息在新的目标上调用。这个备援接受者是相同的原理。可以为改变消息的内容，参数。使其可以被目标对象接受。也可以更换选择子等。

### 消息转发全流程：

![消息转发全流程](http://upload-images.jianshu.io/upload_images/449095-402f05427a489f4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#第四部分：RunTim的常见应用场合