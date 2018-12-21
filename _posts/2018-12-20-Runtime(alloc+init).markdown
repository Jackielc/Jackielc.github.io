---
layout:     post
title:      "alloc和init"
# date:       2018-12-29 19:00:00
subtitle: Runtime源码阅读
author:     "Jackcao"
header-img: "img/post-bg-20181220.jpg"
tags:
    - Runtime
    - iOS
    - Objective-C
---

# 前言
  ***最近在读Runtime的源代码，有很多疑问和心得，特此记录***

<br/>

# 正文

### NSObject 
在`Objective-C`中，除去一些基本的数据类型之外，几乎所有的类都是`NSObject`的子类。继承`NSObject`的子类都可以访问`Runtime`的接口，并赋予其`Objective-C`对象的基本行为和能力。
> The root class of most Objective-C class hierarchies, from which subclasses inherit a basic interface to the runtime system and the ability to behave as Objective-C objects.

### alloc 和 init
初始化一个继承于NSObject的对象时我们一般都会这样做。
```Objective-C
SomeClass *instance = [[SomeClass alloc]init];
```
alloc和init有什么用？Apple解释称
> **alloc**: Returns a new instance of the receiving class. // 返回该类的一个新的实例对象

> **init**: Implemented by subclasses to initialize a new object (the receiver) immediately after memory for it has been allocated. //由子类实现，以便在为新对象(接收方)分配内存之后立即初始化该对象。

但是为什么要这样做呢？其中的细节是什么样子的？事实真的是这样吗？虽说iOS是闭源的，还好Apple开源了Runtime，让我们可以一探究竟。

### Objective-C Runtime
什么是[Runtime](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)？Apple介绍的很清楚

> Describes the macOS Objective-C runtime library support functions and data structures.

> The Objective-C runtime is a runtime library that provides support for the dynamic properties of the Objective-C language, and as such is linked to by all Objective-C apps. Objective-C runtime library support functions are implemented in the shared library found at /usr/lib/libobjc.A.dylib.

**一句话总结：Runtime是Objective-C 面向对象和动态机制的基石。**

### 下载和编译Runtime
`Runtime`目前是开源的，开发者可以自由下载。进入[Apple Open Source](https://opensource.apple.com/source/)，按下`cmd`+`f`并输入`objc4`，然后定位并点击进入目录，will see it
![post-runtime-files](/img/in-post/in-post-2018/post-runtime-files.png)

尾数越大的表明版本越新，不过我在这里使用的是`objc4-723`版本，虽然Apple有压缩包可以[下载](https://opensource.apple.com/tarballs/objc4/)，但是下载的源代码直接进行
编译是不通过的，编译的整个过程很麻烦，需要补充很多文件，如果图省事可以直接去Github上寻找已经可以编译的版本。

### 探究调用过程
新建名为`SomeClass`的`NSObject`子类，在`main.m`中初始化，打上断点然后build
```objective-c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        SomeClass *instance = [[SomeClass alloc]init]; //break point
        NSLog(@"%@",instance);
    }
    return 0;
}
```
此时进入`NSObject.mm`文件中，我们会看到`alloc`的实现,`alloc`只是简单的调用了`_objc_rootAlloc`函数
```objective-c
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```
`_objc_rootAlloc`的实现为
```objective-c
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```
重点是`callAlloc`的实现
```objective-c
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```
`callAlloc`方法中有三个参数,`cls`为当前的调用类，`checkNil`为判空，`allocWithZone`表示是否实现了`allocWithZone`，`=false`代表默认是没有实现的。`NSZone`其实已经在Objective-C中被弃用了，为什么会被保留下来，Apple解释称`allocWithZone`的存在是由于历史原因才会被保留下来。
```
if (slowpath(checkNil && !cls)) return nil;
```
进入方法内部，会发现一个判空的宏，叫做`slowpath`，计算机术语叫做慢路径，表示条件很可能为`false`，让编译器做优化，以此提升性能，虽然是一个很小的优化，但是对于alloc这样在应用中会被大量调用的方法来说，这样做还是很值得的。

然后是条件编译
```
#if __OBJC2__

   ...

#endif
```
表示当前的`Objective-C`版本为2.0才能进入，一般情况下在我们的开发环境中`Objective-C`的版本都是2.0。
条件成立，然后进入
```
fastpath(!cls->ISA()->hasCustomAWZ())
```
`fastpath`为快路径，表示结果大概率为真，`cls->ISA()->hasCustomAWZ()`表示当前类是否存在`alloc`和`allocWithZone`的实现，根据其中的注释，这个判断操作返回的结果大概率为`true`。

紧接着进入
```
if (fastpath(cls->canAllocFast()))
```
从字面意思来讲，这个方法应该是判断当前类是否可以快速地进行`alloc`操作，不过当我进入`canAllocFast`的方法内部时，我发现这个方法永远返回的是`false`，所以最后会进入`else`分支
```
// Has ctor or raw isa or something. Use the slower path.
    id obj = class_createInstance(cls, 0);
    if (slowpath(!obj)) return callBadAllocHandler(cls);
    return obj;
```
开始创建新的实例，(`isa`指针先放在这里，以后再去探究。)
```
/***********************************************************************
* class_createInstance
* fixme
* Locking: none
**********************************************************************/
id 
class_createInstance(Class cls, size_t extraBytes)
{
    //---
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();

    size_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) {
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    return obj;
}、

```
创建实例和分配空间的操作都在`_class_createInstanceFromZone`方法中进行。
```
 if (!cls) return nil;

    assert(cls->isRealized()); // 防止并发实现

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cls->hasCxxCtor(); // 是否有C++构造函数
    bool hasCxxDtor = cls->hasCxxDtor(); // 是否有C++析构函数
    bool fast = cls->canAllocNonpointer(); //是否需要初始化isa指针。如果在64位机器上运行，会返回true

```

然后获取对象大小
```
cls->instanceSize(extraBytes); // extraBytes为额外需要的bytes
```
`instanceSize`的内部实现会进行字节对齐，而且会对小于16`bytes`的对象，强制给予16`bytes`。因为`Apple`称
> ***CF(CoreFoundation) requires all objects be at least 16 bytes.*** //CoreFoundation 要求所有的对象大小至少为16字节

我们不需要`NSZone`而且`fast`为`ture`，所以我们会直接调用`calloc`进行内存分配,并且初始化`isa`指针
```
if (!zone  &&  fast) {
    obj = (id)calloc(1, size);
    if (!obj) return nil;
    obj->initInstanceIsa(cls, hasCxxDtor);
} 
```
值得一提的是在`initInstanceIsa`方法中会判断当前类是否支持`TAGGED POINTERS`(除去一些外部因素，64位机器都支持)：
1. ***Tagged Pointer*** 是Apple在2013年推出的技术。因为在64位系统架构下，指针变量由32位增加到64位，这意味着对于一些对象(`NSNumber`、`NSDate`等)来说可能会存在存储浪费的问题，所以Apple决定将一些比较小的数据直接存储在指针变量中，这些指针变量就叫做`TAGGED POINTERS`。
2. ***Tagged Pointer*** 的值不再是地址，而是真正的值。但是它并不是一个对象，而是一个变量，而且内存不在堆中。

```
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); //不允许当前对象是‘Tagged Pointer’，因为TaggedPointer是一个伪对象不允许拥有isa指针
    
    if (!nonpointer) { //是否开启TAGGED POINTERS
        isa.cls = cls;
    } else {
        assert(!DisableNonpointerIsa); //xcode中禁用了‘TAGGED POINTERS’环境变量
        assert(!cls->instancesRequireRawIsa());// 实例不需要isa指针

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        assert(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE; //赋值给isa联合体
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```
如果分配内存失败则会调用`callBadAllocHandler`函数并抛出异常。如果当前类有自定义的`allocWithZone`的话就会去调用分支
```objective-c
    if (allocWithZone) {
        return [cls allocWithZone:nil];
    }
    return [cls alloc]; // 如果所有条件都不满足，那就开启死循环
```
到此，一个对象就成功地分配到了内存。然后就会调用`init`函数
```objective-c

- (id)init {
    return _objc_rootInit(self);
}

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}

```
是的，Apple直接给我们返回了对象本身，什么都没做，这时候我们就要思考这个函数存在的意义了。

### init函数
我们回头再来看Apple说过的话
> ***init***: Implemented by subclasses to initialize a new object (the receiver) immediately after memory for it has been allocated. //由子类实现，以便在为新对象(接收方)分配内存之后立即初始化该对象。

通过`Runtime`的源代码我们可以知道，`NSObject`在`init`函数中并没有做任何操作，它只是返回了`self`。其实`Apple`在开发文档中说的很明白，`init`函数是留给子类重载的，子类可以在`init`里做一些初始化的操作，比如初始化一些变量、对象...来给实例一些默认的行为和能力。因为子类可以自定义`init`函数的实现，所有在某些情况下我们也可以在`init`函数中返回一个替代的实例，也可以在因为某些原因无法创建实例而返回空值，且不需要抛出异常，但是在`init`函数中必须invoke父类的`init`函数，以此来保证子类可以正确的初始化实例。

### new函数
一个`NSObject`的子类去初始化实例时还有另外一种写法，就是invoke `new`函数，
```objective-c
 SomeClass *instance = [SomeClass new]
```
在`Runtime`中我们可以发现，`new`函数只是`alloc`和`init`函数的联合调用，和默认的构造函数并无二致。
```objective-c
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```
但是我们也可以联想到一些问题，那就是如果子类自定义了构造函数，并在自定义的构造函数中规定了一些能力和行为，我们这时如果使用`new`函数来进行初始化工作，那么问题就出现了：`new`函数内部调用的还是`init`函数，而并不是子类自定义的构造函数，那么就会导致用`new`函数生成的实例并不具备这些能力和行为，必然会导致一系列的问题产生。

### 对象内存
![post-data-bytes](/img/in-post/in-post-2018/post-data-bytes.png)
我们初始化得到的`NSObject`对象其实是一个指针，指针指向对象实际存在的内存，在64位架构下，指针大小为8`bytes`，所以一个NSObjec的实例对象实际大小为8`bytes`，而在上面的源码中我们可以看到，对于小于16`bytes`的对象来说，Apple会强制分配给16`bytes`的内存，这就解释了为什么下面两段代码返回结果不同的原因
```
NSObject *objc = [[NSObject alloc] init];
NSLog(@"objc对象实际需要的内存大小: %zd", class_getInstanceSize([objc class]));
NSLog(@"objc对象实际分配的内存大小: %zd", malloc_size((__bridge const void *)(objc)));

//objc对象实际所需的内存大小: 8
//objc对象实际占用的内存大小: 16
```
其实我们可以手动推算出自定义类的内存布局的内存占用，新建一个`NSObject`的子类`SomeClass`，并转换为`C++`代码，如下：
```objective-c
@interface SomeClass: NSObject
{
    int count;
}
@end

// 转换后得到的C++代码
struct SomeClass_IMPL {
    struct NSObject_IMPL NSObject_IVARS; //isa指针
    int count;
};

struct NSObject_IMPL {
    Class isa; //指向struct objc_class结构体类型的指针
};

```
通过结构体`SomeClass_IMPL`我们可以看到，该结构体有两个成员变量:一个`isa`指针和`int`型变量，在64位架构下`isa`指针占用8`bytes`、`int`型占用4`bytes`。所以最终结果为8+4=12`bytes`
，所以`SomeClass`的实例需要12`bytes`的内存，由于12`bytes`小于16`bytes`，所以系统最后会分配给该对象16`bytes`，然后我们对比下系统输出：

```objective-c
SomeClass *instance = [[SomeClass alloc] init];
NSLog(@"objc对象实际需要的内存大小: %zd", class_getInstanceSize([instance class]));
NSLog(@"objc对象实际分配的内存大小: %zd", malloc_size((__bridge const void *)(instance)));

//objc对象理论所需的内存大小: 16
//objc对象实际占用的内存大小: 16
```
然而`class_getInstanceSize`输出与我们预期的12`bytes`不符，为什么？因为字节对齐
> **字节对齐** :结构体大小需是最大成员变量大小的整数倍。
在结构体`SomeClass_IMPL`中`isa`指针为最大成员变量，根据字节对齐的规则，对象所需内存为`isa`指针内存大小的倍数，即16`bytes`。

如果我们再增加一个`double`型的成员变量。
```objective-c
@interface SomeClass: NSObject
{
    int count;
    double width;
}
@end

// 转换为C++代码
struct SomeClass_IMPL {
    //isa指针
    struct NSObject_IMPL NSObject_IVARS; 
    int count;
    double width;
};

struct NSObject_IMPL {
    //指向struct objc_class结构体类型的指针

    Class isa; 
};
```
`double`型在64位架构下占用8`bytes`，12+8=20`bytes`，由于结构体`SomeClass_IMPL`中最大的成员变量为`isa`指针，然后根据字节对齐规则，得出`SomeClass`的实例对象所需24`bytes`，占用内存24`bytes`，然后来验证一下结果
```objective-c
SomeClass *instance = [[SomeClass alloc] init];
NSLog(@"objc对象实际需要的内存大小: %zd", class_getInstanceSize([instance class]));
NSLog(@"objc对象实际分配的内存大小: %zd", malloc_size((__bridge const void *)(instance)));

//objc对象理论所需的内存大小: 24
//objc对象实际占用的内存大小: 32
```

然而系统结果还是和我们的预期不符，这又是为什么？

> iOS中的`malloc`函数分配内存空间时，是根据`bucket`来分配的。`bucket`的大小是16的倍数。

可以看出系统是按16的倍数来分配对象的内存大小的。由于24并不是16的倍数，所以系统取值32，固分配内存32`bytes`，这就我什么系统输出和我们预期不符的原因。

### So
**`alloc`函数负责分配内存并返回地址给指针，`init`则更多的关注于初始化实例的行为和能力，对象内存是根据CF Require、字节对齐、bucket等规则来分配的。**
