---
layout:     post
title:      PartitionAlloc
date:       2020-09-15
author:     Hope
header-img: img/google-chrome-logo.jpg
catalog: true
tags:
    - Chromium
    - Memory
---
## 简介
`PartitionAlloc` 是一个为安全优化过的、高效的内存分配器
- 快
- 安全

## 特点
- 每个`Partition`的都是独立，且`Partition`中所拥有的内存，在释放相对应的物理内存后，虚拟内存仍被保留。因此`Partition`的虚拟内存是不会被别的`Partition`所使用(安全性)
- 每个`Partition`拥有一个`PartitionBucket`数组，`PartitionBucket`包含着大小相近的对象
- `PartitionAlloc`的allocation 和 deallocation 操作都只有hot 或者 fast 两个分支(高效)
- `PartitionAlloc` 很多方法都是inline的
- 在单线程中，支持无锁操作。在多线程中，使用高效的自旋锁来进行同步

## PartitionRoot 和 PartitionRootGeneric
`PartitionAlloc`有两种实现，分别是`PartitionRoot`和`PartitionRootGeneric`，但是我们不会直接使用`PartitionRoot`和`PartitionRootGeneric`，而是使用相对应的`SizeSpecificPartitionAllocator<size>`和`PartitionAllocatorGeneric`。因为`SizeSpecificPartitionAllocator<size>`和`PartitionAllocatorGeneric`还通过`PartitionAllocMemoryReclaimer`来进行内存的回收。下面列出这两种实现的特点。
### PartitionRoot
- Allocation和Free 一个`Partition`必须要在同一个线程（不支持多线程，所以不需要锁，速度快）
- `PartitionRoot`在`init()`时，需指定一个最大值MAX_SIZE，后序在该`PartitionRoot`上申请的空间，都应该小于该对象的MAX_SIZE，否则会申请失败
- 所申请的大小必须与系统指针大小所对齐

### PartitionRootGeneric
- 支持多线程 Allocation和Free 一个`Partition`，采取自旋锁来进行同步
- 可以申请任意大小的空间，无大小限制

## Blink中的PartititionAlloc
Blink中的内存分配只推荐使用`PartitiionAlloc`或者`OilPan`。其中不同类型的对象，使用不同类型的`PartitionAlloc`.

* LayoutObject partition：主要分配layou相关对象，如LayoutObject、PaintLayer、双向字体BidiCharacterRun等对象，使用`PartitionRoot`
* Buffer partition：主要分配变长对象或者是内容可能会被用户脚本篡改的对象，如Vector、HashTable、ArrayBufferContent、String等容器类对象。使用`PartitionRootGeneric`
* ArrayBuffer partition：主要分配`ArrayBufferContents`对象
* FastMalloc partition：主要分配除其他三种类型之外的比如被标记为USING_FAST_MALLOC的类对象，Blink中大量功能逻辑的内部对象都归于此分区

通过使用宏`USE_ALLOCATOR`来重写类中的`operator new`、`operator delete`、`operator new[]`、`operator delete[]`等方法来使用`PartitionAlloc`相对应的方法。
``` C++
#define USE_ALLOCATOR(ClassName, Allocator)                       \
 public:                                                          \
  void* operator new(size_t size) {                               \
    return Allocator::template Malloc<void*, ClassName>(          \
        size, WTF_HEAP_PROFILER_TYPE_NAME(ClassName));            \
  }                                                               \
  void operator delete(void* p) { Allocator::Free(p); }           \
  void* operator new[](size_t size) {                             \
    return Allocator::template NewArray<ClassName>(size);         \
  }                                                               \
  void operator delete[](void* p) { Allocator::DeleteArray(p); }  \
  void* operator new(size_t, NotNullTag, void* location) {        \
    DCHECK(location);                                             \
    return location;                                              \
  }                                                               \
  void* operator new(size_t, void* location) { return location; } \
                                                                  \
 private:                                                         \
  typedef int __thisIsHereToForceASemicolonAfterThisMacro
```

## 使用示例
``` C++
#include <string.h>

#include "base/allocator/partition_allocator/partition_alloc.h"
#include "base/logging.h"
#include "third_party/blink/renderer/platform/wtf/allocator/partitions.h"

int main(int argc, char const* argv[]) {
  const char kPartitionsDemo[] = "PartitionsDemo";
  auto copy_and_log = [](char* p, char const* str, int length,
                         std::string tag) {
    memcpy(p, str, length);
    LOG(INFO) << "the value of " << tag << " is " << p;
  };

  // 通过WTF::Partitions调用
  WTF::Partitions::Initialize();
  char* fast_ptr =
      (char*)WTF::Partitions::FastMalloc(sizeof(kPartitionsDemo), "char");
  copy_and_log(fast_ptr, kPartitionsDemo, sizeof(kPartitionsDemo), "fast_ptr");
  WTF::Partitions::FastFree((void*)fast_ptr);

  char* buffer_ptr = (char*)WTF::Partitions::BufferPartition()->Alloc(
      sizeof(kPartitionsDemo), "char");
  copy_and_log(buffer_ptr, kPartitionsDemo, sizeof(kPartitionsDemo),
               "buffer_ptr");
  WTF::Partitions::BufferPartition()->Free((void*)buffer_ptr);

  // 直接使用PartitionAllocatorGeneric 和 SizeSpecificPartitionAllocator<size>
  base::PartitionAllocatorGeneric partition_allocator_generic;
  base::SizeSpecificPartitionAllocator<1024> size_specific_partition_allocator;
  partition_allocator_generic.init();
  size_specific_partition_allocator.init();

  char* allocator_generic_ptr =
      (char*)partition_allocator_generic.root()->Alloc(sizeof(kPartitionsDemo),
                                                       "char");
  copy_and_log(allocator_generic_ptr, kPartitionsDemo, sizeof(kPartitionsDemo),
               "allocator_generic_ptr");
  partition_allocator_generic.root()->Free(allocator_generic_ptr);
  
  // 因为PartitionRoot 所申请的大小必须与系统指针大小所对齐
  const size_t pointer_size = sizeof(void*);
  int i = 1;
  while (i*pointer_size<sizeof(kPartitionsDemo))
  {
    ++i;
  }
  char* size_specific_partition_allocator_ptr =
      (char*)size_specific_partition_allocator.root()->Alloc(
          i*pointer_size, "char");
  copy_and_log(size_specific_partition_allocator_ptr, kPartitionsDemo, sizeof(kPartitionsDemo)+1,
               "size_specific_partition_allocator_ptr");
  partition_allocator_generic.root()->Free(size_specific_partition_allocator_ptr);

  return 0;
}
```
## 参考文档
- [demo_partition_alloc.cc](https://github.com/YuWeiCong/chromium_demo/blob/master/memory/demo_partition_alloc.cc)
- [Blink内存分配器](https://blog.csdn.net/woweiwokuang0000/article/details/50859029)
- [PartitionAlloc Design](https://chromium.googlesource.com/chromium/src/+/master/base/allocator/partition_allocator/PartitionAlloc.md)
- [partition_alloc.h](https://source.chromium.org/chromium/chromium/src/+/master:base/allocator/partition_allocator/partition_alloc.h)