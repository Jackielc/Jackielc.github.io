---
layout:     post
title:      "isa与meta class"
subtitle:   Runtime源码阅读
# date:       2018-12-29 19:00:00
author:     "Jackcao"
header-img: "img/post-bg-20181227.jpg"
tags:
    - Runtime
    - iOS
    - Objective-C
---
 
# 前言
***之前在`Runtime`中发现每个对象中都包含`isa`指针，但是它有什么用？结构又是怎样的？***

> [- Hamster Emporium:《Classes and metaclasses》](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)<br/>
> [- Matt Gallagher:《What is a meta-class in Objective-C?》](https://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)<br/>
> [- draveness:《从 NSObject 的初始化了解 isa》](https://github.com/draveness/analyze/blob/master/contents/objc/从%20NSObject%20的初始化了解%20isa.md)

<br/>
<br/>


# 正文
### Class
`Objective-C`是基于对象的动态语言，每一个对象都是对象所属类的实例，类描述了实例对象的以下行为：
* `allocation size`
* `ivars type`
* `layout`
* `SEL`
* `Methods`

我们在实际开发中也都知道`instance methods`和`class methods`，即实例方法和类方法，实例方法只能由类的实例对象调用，类方法只能由类调用，那你有没有想过，实例方法和类方法都存储在哪里？
现在我们可以来考虑一下，如果一个对象所有能执行的方法都存在对象中，而且这个对象的方法数量众多，那么所有该类型的对象将会占用很大的内存，这样做分布式的存储方式显然是不合理的而且也是没有必要的。**其实我们可以集中式地统一管理对象的方法：那就是存在对象所属的类中。**

### Meta Class
更详细地来说，实例方法和类方法分别存放在`Class`和`Meta Class`中，`Meta Class`即为元类。`Class`不仅是一个对象，还是`Meta Class`的实例对象，而且重要的是`Meta Class`和`Class`的关系类似于`Class`和`Instances`的关系，比如当`instances`调用实例方法时，`Class`会从自身维护的实例方法列表中去寻找并决定调用该实例方法，同理，当调用类方法时，`Meta Class`会从自身维护的类方法列表中寻找并决定调用该类方法。

### isa 指针
在`Objective-C`中每个继承于`NSObject`的子类的实例对象都会存在一个`Class`类型的成员变量，叫做`isa`。把它们的所有关联都列出来：
```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

typedef struct objc_class *Class;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    ...
}

struct objc_object {
private:
    isa_t isa;
  
    ...
}

```
我们可以从这段代码中发现，`Class`是`objc_class`结构体指针，`objc_class`继承于`objc_object`。还有一个重要的发现：关键字`id`其实是`objc_object`结构体指针，我们在开发中可以使用`id`来声明一个对象，这就验证了 **`Class`其实也是`Objective-C`对象** 的结论。接下来我们来把`Class`结构体的结构简化为：
```
{
    isa_t isa;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}
```
由此我们可以看出所有的类的实例对象都应该包含这样的一个结构。类也是一个对象，而且是`Meta Class`的对象。在`Objective-C`中，所有的对象(类)都是`objc_object`结构体指针，所以所有的对象(类)都应包含一个`isa_t`结构体指针，而该指针指向的是对象所属的类(元类)，当对象(类)方法被调用时，对象(类)通过持有的`isa_t`指针去类(元类)中寻找方法的实现，也会通过持有的`super_class`指针去父类(父元类)中寻找方法的实现，以完成在继承关系中的链式查找。我还有一个疑问，那`Meta Class`是否也存在继承链呢？答案是存在的，就像普通的对象和类一样，`Meta Class`也是一个对象，也存在继承链。下面这张经典的图表很直观地阐述了实例对象和类、类和元类之间的关系：
![runtime-object-chain](/img/in-post/in-post-2018/runtime-object-chain.png)

### isa_t指针

```
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if SUPPORT_PACKED_ISA
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif
};
```
从代码中我们发现，`isa_t`是一个`union`类型的结构体，`union`类型的结构体与普通结构体最大的不同就是它是共用存储空间的，而且在不同的架构下`isa_t`的内存布局也有比较大的差异，`SUPPORT_PACKED_ISA`规定在`32`位架构、`win32`、`macOS`模拟器环境中`isa_t`的结构体不可设置，此时访问对象的`isa`指针，会直接返回一个指向`cls`的指针。`Runtime`目前是在`macOS`上编译运行的，即我们现在只讨论`__x86_64__`架构下的`isa_t`指针。

```
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) 
{ 
    if (!indexed) {
        isa.cls = cls;
    } else {
        isa.bits = ISA_MAGIC_VALUE; // 0x001d800000000001ULL
        isa.has_cxx_dtor = hasCxxDtor;
        isa.shiftcls = (uintptr_t)cls >> 3;
    }
}
```
上面的代码在初始化一个`isa_t`指针，并且对其设置值，`ISA_MAGIC_VALUE`是十六进制，转换为二进制并补齐到64位为`0000000000011101100000000000000000000000000000000000000000000001`，如下图
![runtime-objc-isa-isat-bits](/img/in-post/in-post-2018/runtime-objc-isa-isat-bits.png)

实际上只设置了`indexed`、`magic`这两部分的值。`isa_t`中其它`bit`的含义如下：

* **indexed**: 表示`isa_t`的类型(是否存在结构体)。主要的区别就是`arm64`、`__x86_64__`架构和其他架构的差别。简单地来说就是用于区分`iPhone`迁移到`64`位系统之前时`isa`的类型(0)与迁移之后的类型(1)。
* **has_assoc**: 对象含有或者曾经含有关联引用。
* **has_cxx_dtor**: 表示当前对象是否有`C++`或者`ObjC`的析构。
* **shiftcls**: 保存关于类的指针。
* **magic**: 用于调试器判断当前对象是真的对象还是没有初始化的空间。
* **weakly_referenced**: 对象被指向或者曾经指向一个`ARC`的弱变量。
* **deallocating**: 对象是否正在释放内存。
* **has_sidetable_rc**: 如果`extra_rc`计数满了，不够用用存放计数了，`has_sidetable_rc`就会为1
* **extra_rc**: 对象的引用计数超过1，会存在这里面，如果引用计数为3，`extra_rc`的值就为2。`extra_rc`只有8位，所以它最多能计数到255。

# 总结
`isa`指针更偏向于底层，并没有做很深入的探究，目前只需要了解到`isa`指针的基本概念和结构就已适用。`meta class`的更多实现细节已经被`Apple`所隐藏，我们只能看到它的外在表现，而更重要的内部实现我们无从所知。