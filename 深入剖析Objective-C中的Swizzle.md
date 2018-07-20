---
title: 深入剖析Objective-C中的Swizzle
date: 2017-12-15 09:40:04
tags:	iOS
---

有时候为了达到一些特殊的需求，我们会在运行时期交换两个方法的实现，用我们自己的方法替换原始方法。在OC运行时期，OC方法被表示为一个名为Method的C结构`typedef struct objc_method *Method`,在runtime.h文件中，这个结构体如下：

```objective-c
struct objc_method {
    SEL _Nonnull method_name                               OBJC2_UNAVAILABLE;
    char * _Nullable method_types                          OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                OBJC2_UNAVAILABLE;
} 
```

method_name是函数的选择器,method_type是参数和返回值类型编码的c字符串,method_imp是指向实际函数的函数指针。可以通过下面两个方法访问到object_method对象:

```objective-c
Method class_getClassMethod(Class aClass, SEL aSelector);
Method class_getInstanceMethod(Class aClass, SEL aSelector);
```

通过Method结构体可以访问更改底层的实现。实际上，我们完成swizzle操作就是将两个方法的IMP进行了一次交换。如何实现两个方法IMP的交换呢？我将列举两种方式，后面将对两种方式的进行比较。一种简单的实现方式就是通过：

```objective-c
void method_exchangeImplementations(Method m1, Method m2);
```

通过`class_getClassMethod`或(`class_getInstanceMethod`)获取到method对象，通过调用`method_exchangeImplementations`就可实现两个方法的IMP的交换，执行完方法交换后，他们的objc_method结构体发生如下变化：

![](https://ws3.sinaimg.cn/large/006tNc79ly1fmhgb69vnvj31620lsgp1.jpg)

在代码调试的过程中，发现在一个问题，使用了`method_exchangeImplementations`实现方法交换之后，两个方法的内部`_cmd`与原来的方法名称进行了传递:

```objective-c
+ (void)method1{
  // _cmd isEqueToString 'method2'
}

// at the catogary
+ (void)method2{
  [self method2];
  //_cmd isEqueToString 'method1'
}
```

显而易见的是这种方式是有弊端的，如果一个函数实现依赖_cmd和方法名称，这种方式会有想不到的后果产生。下面我们来看看第二种swizzle的实现。

第二种实现swizzle的方式是写一个类似IMP的C函数，IMP的定义如下：

```objective-c
void (*)(id,SEL,...)
```

类似IMP的C函数：

```objective-c
void _swizzle_method2(self,_cmd){
  
}
```

我们可以将这个函数作为一个IMP来使用，通过`method_setImplementation`方法，将这个IMP替换掉原始方法的IMP:

```objective-c
IMP originImp = method_setImplementation(method1,(IMP)_swizzle_method2);
```

originImp是原始方法的IMP函数指针。如果想调用原始方法：

```objective-c
originImp(self,_cmd);
```

重要的代码实现：

```objective-c
//dic全局的字典对象，用来保存原来的方法实现
Method method = class_getInstanceMethod(self.class, @selector(method1));
IMP originImp = method_setImplementation(method, (IMP)_swizzle_method2);
if (!dic) {
    dic = [NSMutableDictionary new];
}
[dic setObject:[NSValue valueWithPointer:originImp] forKey:@"method"];


//交换原始方法IMP的C函数
void _swizzle_method2(id self,SEL _cmd){
    NSValue *value = [dic valueForKey:@"method"];
    if (value) {
        IMP originImp = [value pointerValue];
        ((void (*)(id,SEL))originImp)(self,_cmd);
    }
}

//原来的方法
- (void)method1
{
  //_cmd isEqueToString "method1" 
}
```

这里我们可以发现，原始方法method1现在函数内的_cmd和原始方法名是相等的。因为OC函数的调用都会返回两个隐藏的参数`(self, _cmd)`,如果我们忽略了这个特性，很可能在日常开发中，造成不必要的麻烦。很显然第二种方式实现swizzle更优雅。

