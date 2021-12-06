---
title: 基于objc4的Runtime原理解读（一）
date: 2021-11-30 22:45:52
tags: 
- iOS
- objective-c
- runtime
categories: iOS
---
## 前言
学习objective-c，runtime一直是一个绕不过去的话题，为什么iOS系统开发选择使用oc而不是c++呢？原因就是runtime这一大利器。

> The Objective-C language defers as many decisions as it can from compile time and link time to runtime. Whenever possible, it does things dynamically. This means that the language requires not just a compiler, but also a runtime system to execute the compiled code. The runtime system acts as a kind of operating system for the Objective-C language; it’s what makes the language work.

Runtime有两个版本，一个Legacy版本 ，一个Modern版本。Legacy版本使用的是Objective-c 1，也是大多数人所了解过的runtime版本，而Modern版本则使用了Objective-c 2.0。
- iPhone applications and 64-bit programs on OS X v10.5 and later use the modern version of the runtime.
- Other programs (32-bit programs on OS X desktop) use the legacy version of the runtime.

网上有很多资料介绍的都是Legacy版本下的runtime源码，但实际上现在已经几乎没有任何程序基于此版本的runtime执行了。

接下来的内容，我将以苹果最新开源的[objc4版本的源码](https://opensource.apple.com/source/objc4/)，对runtime相关知识进行介绍。

## 从类说起

### objc_object

```
<objc-private.h>

typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;
}


union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;

private:
    // Accessing the class requires custom ptrauth operations, so
    // force clients to go through setClass/getClass by making this
    // private.
    Class cls;

public:
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };

}
```
objc_object就是oc中最基础的类型，id类型其实也是一个指向objc_object类型的指针。可以看到，objc_object结构体中只有isa_t类型的isa指针。

而isa_t则是一个共用体，里面不止存在指向Class类型的指针cls，还存在bits这样一个指针，使用位域技术存储了更多信息。bits指针根据不同的架构，存储了不同的内容，下面举arm64架构为例:
```
#     define ISA_MASK        0x0000000ffffffff8ULL
#     define ISA_MAGIC_MASK  0x000003f000000001ULL
#     define ISA_MAGIC_VALUE 0x000001a000000001ULL
#     define ISA_HAS_CXX_DTOR_BIT 1
#     define ISA_BITFIELD                                                      \
        uintptr_t nonpointer        : 1;                                       \
        uintptr_t has_assoc         : 1;                                       \
        uintptr_t has_cxx_dtor      : 1;                                       \
        uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
        uintptr_t magic             : 6;                                       \
        uintptr_t weakly_referenced : 1;                                       \
        uintptr_t unused            : 1;                                       \
        uintptr_t has_sidetable_rc  : 1;                                       \
        uintptr_t extra_rc          : 19
#     define RC_ONE   (1ULL<<45)
#     define RC_HALF  (1ULL<<18)
```
shiftcls中存储着Class、Meta-Class对象的内存地址信息，对象的isa指针需要同ISA_MASK经过一次&（按位与)运算才能得出真正的Class对象地址。

ISA_MASK：  0x0000000ffffffff8ULL
二进制：111111111111111111111111111111111000

可以看出ISA_MASK最后三位的值为0，那么任何数同ISA_MASK按位与运算之后，得到的最后三位必定都为0，因此任何类对象或元类对象的内存地址最后三位必定为0，转化为十六进制末位必定为8或者0。这样是因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的0，对齐后可以提高代码运行的性能。

![isa_t](isa_t.png)

### Class

继续查看，我们可以看到，Class其实是一个指向objc_class结构体的指针。
```
<objc.h>

typedef struct objc_class *Class;
```

而objc_class结构体继承自objc_object。

```
<objc-runtime-new.h>

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() const {
        return bits.data();
    }
}
```
至此，我们可以得到这样一张关系图：

![class](class.png)

从图中我们可以看到，<b>类其实也是一个对象</b>，除了拥有一个指向类类型的isa指针外，还拥有父类指针、方法缓存和存储类数据的bits指针三个属性。

下面看一下cache_t的结构，最新版的runtime源码中，_bucketsAndMaybeMask代替了之前的buckets_t指针，作为一个散列表，存储方法的imp指针。

每次去寻找方法调用时，会先从对象的方法缓存中进行寻找，提高效率。

```
<objc-runtime-new.h>

struct cache_t {
private:
    explicit_atomic<uintptr_t> _bucketsAndMaybeMask;
    union {
        struct {
            explicit_atomic<mask_t>    _maybeMask;
#if __LP64__
            uint16_t                   _flags;
#endif
            uint16_t                   _occupied;
        };
        explicit_atomic<preopt_cache_t *> _originalPreoptCache;
    };
    // _bucketsAndMaybeMask is a buckets_t pointer
    // _maybeMask is the buckets mask
}


struct bucket_t {
private:
    // IMP-first is better for arm64e ptrauth and no worse for arm64.
    // SEL-first is better for armv7* and i386 and x86_64.
#if __arm64__
    explicit_atomic<uintptr_t> _imp;
    explicit_atomic<SEL> _sel;
#else
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif
}
```

class_data_bits_t同样采用了位域技术，注释中我们可以看出，class_data_bits_t相当于class_rw_t * 加上自定义的rr/alloc标志。

通过data方法取到class_rw_t。

```
<objc-runtime-new.h>

#define FAST_DATA_MASK          0x00007ffffffffff8UL

struct class_data_bits_t {
    friend objc_class;

    // Values are the FAST_ flags above.
    uintptr_t bits;

public:
    class_rw_t* data() const {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }

    const class_ro_t *safe_ro() const {
        class_rw_t *maybe_rw = data();
        if (maybe_rw->flags & RW_REALIZED) {
            // maybe_rw is rw
            return maybe_rw->ro();
        } else {
            // maybe_rw is actually ro
            return (class_ro_t *)maybe_rw;
        }
    }
}
```
class_rw_t内包含了类的方法列表、属性列表和协议列表，一如其名，是可读写的（read-write)。

```
<objc-runtime-new.h>

struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif

public:
const class_ro_t *ro() const {
        auto v = get_ro_or_rwe();
        if (slowpath(v.is<class_rw_ext_t *>())) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->ro;
        }
        return v.get<const class_ro_t *>(&ro_or_rw_ext);

    }

    const method_array_t methods() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->methods;
        } else {
            return method_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseMethods()};
        }
    }

    const property_array_t properties() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->properties;
        } else {
            return property_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseProperties};
        }
    }

    const protocol_array_t protocols() const {
        auto v = get_ro_or_rwe();
        if (v.is<class_rw_ext_t *>()) {
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->protocols;
        } else {
            return protocol_array_t{v.get<const class_ro_t *>(&ro_or_rw_ext)->baseProtocols};
        }
    }
}
```
在class_rw_t内还存储了class_ro_t指针。
```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    union {
        const uint8_t * ivarLayout;
        Class nonMetaclass;
    };

    explicit_atomic<const char *> name;
    // With ptrauth, this is signed if it points to a small list, but
    // may be unsigned if it points to a big list.
    void *baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
}
```
class_ro_t为只读类型，存储了编译期间就确定的类的信息，包括ivar、protocol、property、method等。

class_ro_t则为读写类型，用于存储在运行时动态绑定的类的信息。

在编译期间class_data_bits_t取到的data实际上为class_ro_t，此时flags指针的RW_REALIZED位为0。
![ro](ro.png)

在该类真正被实现后，class_data_bits_t取到data实际上才为class_rw_t类型。
![rw](rw.png)

## 总结

我们可以用下面这张图来进行总结。
![summary](summary.png)

## 参考文稿

https://halfrost.com/objc_runtime_isa_class/#toc-7

https://draveness.me/method-struct/