---
title: Android Binder 子系统学习随笔
date: 2024-12-05 22:07:00
permalink: android-binder-learning-note/
tags: 
  - Android
---

> 其实本文在 2023 年 5 月写下，一直放着没有整理，直到今天。

# Binder Driver

驱动层到底干了什么，binder 框架对应的是 C/S之间的通讯，而 binder 是这中间的桥梁，将发送方的信息传递给接收方。binder 处理了这其中底层的数据交换

- 打开 binder 设备
- mmap 映射内存：此操作将用户控件的内存映射到内核空间中
- 传递客户端与服务端的数据
- 托管线程管理

## flat_binder_object

这是 binder 中用来跨进程传递对象的结构体。

```c
struct flat_binder_object {
	__u32		type;
	__u32		flags;

	union {
		binder_uintptr_t	binder; /* local object */
		__u32			    handle;	/* remote object */
	};
	binder_uintptr_t	cookie;
};
```

其中的 type 为

```c
enum {
    // 用来标记当前传递的是 binder 实体
	BINDER_TYPE_BINDER	    = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
	BINDER_TYPE_WEAK_BINDER	= B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
    // 用来标记当前传递的是 binder 在 driver 中的引用地址
	BINDER_TYPE_HANDLE	    = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
	BINDER_TYPE_WEAK_HANDLE	= B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
	BINDER_TYPE_FD		    = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
};
```

当 Server 把 binder 实体传递给 Client 的时候，Server 携带的 `flat_binder_object.binder` 是 binder 实体在 Server 用户空间的内存，driver 会自动的将此变量存储为 driver 中保存的为 client 创建的 **binder 实体引用**指针，然后再将 `flat_binder_object` 传递给 client，client 才能将此指针填入到 `binder_transation_data` 的 **target.handle** 域中，才能正确的像 Server 发起 IPC 请求。

## 驱动层的线程管理

驱动从最开始设计就为多线程做了支持，线程实际在 C/S 端去创建，驱动作为管理者通知 C/S 端来创建线程、退出线程等。

### 问题

- 线程用完就会退出吗？
- 驱动根据进程有无空闲的线程来决定是否创建线程吗？
- 最大线程数就是就是一个 Server 的最大并发吗？

## ServiceManager

系统中只允许曾在一个 ServiceManager 进程。ServiceManager 只将唯一的服务名称与其在 driver 中的引用号做了映射，其他 client 随用随取，所以他的工作很简单。只有 128K 的缓存空间。

### 问题

- ServiceManager 的线程管理，最大线程数

## debugfs

存储的位置位于 `/sys/kernel/debug/binder`
 
使用方式：

- `/sys/kernel/debug/binder/proc` 查看有哪些进程使用了 binder
- `transaction_log` & `transactions` 文件查看 binder 的通信信息

# C++ 层的 Binder

Framework 层的 binder 将实现分为 native 和 proxy 两端，native 端对应的是服务端的实现，proxy 端对应的是服务的远程接口，使用服务的 client 将会拿到 proxy 端用来发起 RPC 请求。proxy 和 native 内部通过 binder 协议进行中转。client 调用 Foo.bar() 函数的时候，会将函数号、参数信息传递给 binder driver 中，binder 跨进程访问 native 实现的对应函数。实现 RPC 调用。

![binder arch, https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/binder_middleware.png](images/binder_middleware.png)

为什么要这么设计。interface 的设计逻辑是什么。

IInterface 和 IBinder 本质上是同一个层次的抽象。IInterface 承载了服务的借口，IBinder 则标志了这是一个 binder 服务。

IInterface.h 中定义了如下的模版类：

```c++
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};

// ----------------------------------------------------------------------

template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
    explicit                    BpInterface(const sp<IBinder>& remote);

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};
```

我现在认为这两个模版类的作用只是方便开发者实现 BnXXX 和 BpXXX，自动继承自不同的 BBinder 或 BpRefBase。

BnInterface & BpInterface 继承自用户实现的 IXxxInterface 为了约束 binder 的功能接口作为公共的借口提供给 BpXx 和 BnXxx，实现端与代理端从 BnInterface & BpInterface 才开始分叉。

为了满足 Binder 的实现规则，

- 每一个 binder 服务都需要定义一个唯一的标识符，写作 descriptor，通过 getInterfaceDescriptor() 访问
- 为了便于调用者获取到服务借口，公共 Interface 需要定义一个 `android::sp<IXXX> asInterface` 来返回基类对象指针。

IInterface.h 实现了如下的宏，用于开发者快速定义

```c++
// ----------------------------------------------------------------------

#define DECLARE_META_INTERFACE(INTERFACE)                                                         \
public:                                                                                           \
    static const ::android::String16 descriptor;                                                  \
    static ::android::sp<I##INTERFACE> asInterface(const ::android::sp<::android::IBinder>& obj); \
    virtual const ::android::String16& getInterfaceDescriptor() const;                            \
    I##INTERFACE();                                                                               \
    virtual ~I##INTERFACE();                                                                      \
    static bool setDefaultImpl(::android::sp<I##INTERFACE> impl);                                 \
    static const ::android::sp<I##INTERFACE>& getDefaultImpl();                                   \
                                                                                                  \
private:                                                                                          \
    static ::android::sp<I##INTERFACE> default_impl;                                              \
                                                                                                  \
public:

#define __IINTF_CONCAT(x, y) (x ## y)

#ifndef DO_NOT_CHECK_MANUAL_BINDER_INTERFACES

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    static_assert(internal::allowedManualInterface(NAME),               \
                  "b/64223827: Manually written binder interfaces are " \
                  "considered error prone and frequently have bugs. "   \
                  "The preferred way to add interfaces is to define "   \
                  "an .aidl file to auto-generate the interface. If "   \
                  "an interface must be manually written, add its "     \
                  "name to the whitelist.");                            \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)    \

#else

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)    \

#endif

// Macro to be used by both IMPLEMENT_META_INTERFACE and IMPLEMENT_META_NESTED_INTERFACE
#define DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE0(ITYPE, INAME, BPTYPE)                     \
    const ::android::String16& ITYPE::getInterfaceDescriptor() const { return ITYPE::descriptor; } \
    ::android::sp<ITYPE> ITYPE::asInterface(const ::android::sp<::android::IBinder>& obj) {        \
        ::android::sp<ITYPE> intr;                                                                 \
        if (obj != nullptr) {                                                                      \
            intr = ::android::sp<ITYPE>::cast(obj->queryLocalInterface(ITYPE::descriptor));        \
            if (intr == nullptr) {                                                                 \
                intr = ::android::sp<BPTYPE>::make(obj);                                           \
            }                                                                                      \
        }                                                                                          \
        return intr;                                                                               \
    }                                                                                              \
    ::android::sp<ITYPE> ITYPE::default_impl;                                                      \
    bool ITYPE::setDefaultImpl(::android::sp<ITYPE> impl) {                                        \
        /* Only one user of this interface can use this function     */                            \
        /* at a time. This is a heuristic to detect if two different */                            \
        /* users in the same process use this function.              */                            \
        assert(!ITYPE::default_impl);                                                              \
        if (impl) {                                                                                \
            ITYPE::default_impl = std::move(impl);                                                 \
            return true;                                                                           \
        }                                                                                          \
        return false;                                                                              \
    }                                                                                              \
    const ::android::sp<ITYPE>& ITYPE::getDefaultImpl() { return ITYPE::default_impl; }            \
    ITYPE::INAME() {}                                                                              \
    ITYPE::~INAME() {}

// Macro for an interface type.
#define DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                        \
    const ::android::StaticString16 I##INTERFACE##_descriptor_static_str16(                     \
            __IINTF_CONCAT(u, NAME));                                                           \
    const ::android::String16 I##INTERFACE::descriptor(I##INTERFACE##_descriptor_static_str16); \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE0(I##INTERFACE, I##INTERFACE, Bp##INTERFACE)

// Macro for "nested" interface type.
// For example,
//   class Parent .. { class INested .. { }; };
// DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_NESTED_INTERFACE(Parent, Nested, "Parent.INested")
#define DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_NESTED_INTERFACE(PARENT, INTERFACE, NAME)  \
    const ::android::String16 PARENT::I##INTERFACE::descriptor(NAME);                    \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE0(PARENT::I##INTERFACE, I##INTERFACE, \
                                                     PARENT::Bp##INTERFACE)

#define CHECK_INTERFACE(interface, data, reply)                         \
    do {                                                                \
      if (!(data).checkInterface(this)) { return PERMISSION_DENIED; }   \
    } while (false)                                                     \


// ----------------------------------------------------------------------
```

有了这两个宏之后，开发者只要在接口基类（IXXX）头文件中，使用 `DECLARE_META_INTERFACE` 宏便完成了需要的组件的声明。然后在 cpp 文件中使用 `IMPLEMENT_META_INTERFACE` 便完成了这些组件的实现。

## Binder 的初始化

所有使用到 Binder 的进程在开始使用 Binder 进行 IPC 之前都需要打开 binder 设备并且 mmap 映射内存。这部分逻辑因为是进程共有的，所以定义到了 framework 的 ProcessState 类中。

ProcessState 实例为单例，进程唯一，进程只需要打开一次 binder 设备与 mmap 映射。

