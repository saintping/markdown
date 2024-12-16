### 各种垃圾回收器

所有的垃圾回收器都继承自CollectedHeap。

- EpsilonHeap
  只分配不回收，开发环境实验用。
- GenCollectedHeap
  - SerialHeap
    新生代mark-copy，老年代mark-sweep-compact，SafePoint里单线程执行。
  - CMSHeap
    实验性质，只工作在老年代mark-sweep。除扫描GCRoots需要STW，其他大部分的标记和清理都和应用线程一起并行。
- ParallelScavengeHeap
  SerialHeap的优化版本，SafePoint里多线程执行。Java7默认。
- G1CollectedHeap
  取消连续物理的分区，基于Region黑、灰、白三色扫描。可以根据延迟目标，自适应调整回收区域的大小。Java9的默认，Java8可以手动开启。
- ZCollectedHeap
  O(1)空间复杂度；最大延迟10ms；支持4TB内存；最大吞吐量下降15%。Java11的实验版本，Java15默认。

ZGC既支持分代，也支持不分代，总体而言分代的应用范围更广。应该有JEP准备去掉不分代的特性以简化实现。[https://openjdk.org/jeps/490](https://openjdk.org/jeps/490 "https://openjdk.org/jeps/490")

### 回收算法的发展
![garbage-collection.jpg](https://ping666.com/wp-content/uploads/2024/10/garbage-collection.jpg "garbage-collection.jpg")
Serial、Parallel、CMS都是基于分代的回收算法，以吞吐量为优先目标，通过不断的改进并发来降低延迟。
从G1之后的垃圾回收器，包括G1、ZGC、shenandoah都以低延迟为优先目标。

- 分区方案
  为了低延迟，都放弃了分代方案而转为分区方案。以低延迟为目标，自适应选择单次回收多大的区域，小步快跑。
- 代码注入
  在应用的字节码里插入一小部分GC相关的代码。将部分准备工作分散到应用线程里，进一步降低STW的延迟。

本文重点介绍ZCollectedHeap（Z Garbage Collector），官方地址[https://wiki.openjdk.org/display/zgc/Main](https://wiki.openjdk.org/display/zgc/Main "https://wiki.openjdk.org/display/zgc/Main")

### ZGC总览
ZGC的一个重要目标是把延迟控制在10ms以内。一般用户接口的延迟要控制在100ms左右，用户才不会感受到明显的停顿。如果一次GC的停顿是30ms，对用户体验会造成明显伤害。ZGC为了低延迟的目标，使用了很多手段：染色指针（Colored Pointer）、读屏障（Load Barrier）、压缩整理（mark-copy-compact）、整堆扫描部分回收等。

ZGC的堆结构在这里
`jdk\src\hotspot\share\gc\z\zCollectedHeap.hpp`
```cpp
class ZCollectedHeap : public CollectedHeap {
  ZCollectorPolicy* _collector_policy;
  SoftRefPolicy     _soft_ref_policy;
  ZBarrierSet       _barrier_set;
  ZInitialize       _initialize;
  ZHeap             _heap;
  ZDirector*        _director;
  ZDriver*          _driver;
  ZStat*            _stat;
  ZRuntimeWorkers   _runtime_workers;
};
```

### Colored Pointer
ZGC仅支持64系统，就是因为Colored Pointer，它把内存映射成多个镜像的空间，每个4TB。

![zgc-memory-mapping.png](https://ping666.com/wp-content/uploads/2024/09/zgc-memory-mapping.png "zgc-memory-mapping.png")

X86架构支持48位寻址，4TB是42位，上图中使用了4位，还有2位预留。

```cpp
// Metadata types
const uintptr_t   ZAddressMetadataShift         = ZPlatformAddressMetadataShift = 42; // 4TB，X86硬件支持48位寻址
const uintptr_t   ZAddressMetadataMarked0       = (uintptr_t)1 << (ZAddressMetadataShift + 0);
const uintptr_t   ZAddressMetadataMarked1       = (uintptr_t)1 << (ZAddressMetadataShift + 1);
const uintptr_t   ZAddressMetadataRemapped      = (uintptr_t)1 << (ZAddressMetadataShift + 2);
const uintptr_t   ZAddressMetadataFinalizable   = (uintptr_t)1 << (ZAddressMetadataShift + 3);
```

传统的垃圾回收器，给对象打标记是记录在对象头MarkOop里，这需要一次内存访问。而Colored Pointer只需要设置一下指针的某一位。

```cpp
inline uintptr_t ZAddress::marked(uintptr_t value) {
  return address(offset(value) | ZAddressMetadataMarked);
}

inline uintptr_t ZAddress::marked0(uintptr_t value) {
  return address(offset(value) | ZAddressMetadataMarked0);
}

inline uintptr_t ZAddress::marked1(uintptr_t value) {
  return address(offset(value) | ZAddressMetadataMarked1);
}

inline uintptr_t ZAddress::remapped(uintptr_t value) {
  return address(offset(value) | ZAddressMetadataRemapped);
}
```

### Load Barrier
因为ZGC大部分过程（特别是对象转移这一步）是并发的，也就是ZGC工作线程和应用线程是同时工作的。应用线程在工作时肯定会涉及到对象的读写。这就需要一个同步机制来解决冲突，ZGC使用的是内存读屏障。

在Heap加载对象的地方插入一段内存屏障代码。在这里标记指针或者会检查对象是不是被移动了，如果被移动了（Colored Pointer着色），确保重新指向新的对象引用。

这部分代码在
`jdk\src\hotspot\share\gc\z\zBarrier.cpp`
```cpp
inline oop ZBarrier::load_barrier_on_oop(oop o) {
  return load_barrier_on_oop_field_preloaded((oop*)NULL, o);
}

inline oop ZBarrier::load_barrier_on_oop_field(volatile oop* p) {
  const oop o = *p;
  return load_barrier_on_oop_field_preloaded(p, o);
}

inline oop ZBarrier::load_barrier_on_oop_field_preloaded(volatile oop* p, oop o) {
  return barrier<is_good_or_null_fast_path, load_barrier_on_oop_slow_path>(p, o);
}

uintptr_t ZBarrier::load_barrier_on_oop_slow_path(uintptr_t addr) {
  return relocate_or_mark(addr);
}

uintptr_t ZBarrier::relocate_or_mark(uintptr_t addr) {
  return during_relocate() ? relocate(addr) : mark<Strong, Publish>(addr);
}

uintptr_t ZBarrier::relocate(uintptr_t addr) {
  assert(!ZAddress::is_good(addr), "Should not be good");
  assert(!ZAddress::is_weak_good(addr), "Should not be weak good");

  if (ZHeap::heap()->is_relocating(addr)) {
    // Relocate
    return ZHeap::heap()->relocate_object(addr);
  }

  // Remap
  return ZAddress::good(addr);
}

template <bool finalizable, bool publish>
uintptr_t ZBarrier::mark(uintptr_t addr) {
  uintptr_t good_addr;

  if (ZAddress::is_marked(addr)) {
    // Already marked, but try to mark though anyway
    good_addr = ZAddress::good(addr);
  } else if (ZAddress::is_remapped(addr)) {
    // Already remapped, but also needs to be marked
    good_addr = ZAddress::good(addr);
  } else {
    // Needs to be both remapped and marked
    good_addr = remap(addr);
  }

  // Mark
  if (should_mark_through<finalizable>(addr)) {
    ZHeap::heap()->mark_object<finalizable, publish>(good_addr);
  }

  return good_addr;
}
```

### mark-copy-compact
ZGC的垃圾回收也是采用mark-copy-compact模式，整个过程在
`jdk\src\hotspot\share\gc\z\zDriver.cpp`
```cpp
void ZDriver::run_gc_cycle(GCCause::Cause cause) {
  ZDriverCycleScope scope(cause);

  // Phase 1: Pause Mark Start
  {
    ZMarkStartClosure cl;
    vm_operation(&cl);
  }

  // Phase 2: Concurrent Mark
  {
    ZStatTimer timer(ZPhaseConcurrentMark);
    ZHeap::heap()->mark();
  }

  // Phase 3: Pause Mark End
  {
    ZMarkEndClosure cl;
    while (!vm_operation(&cl)) {
      // Phase 3.5: Concurrent Mark Continue
      ZStatTimer timer(ZPhaseConcurrentMarkContinue);
      ZHeap::heap()->mark();
    }
  }

  // Phase 4: Concurrent Process Non-Strong References
  {
    ZStatTimer timer(ZPhaseConcurrentProcessNonStrongReferences);
    ZHeap::heap()->process_non_strong_references();
  }

  // Phase 5: Concurrent Reset Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentResetRelocationSet);
    ZHeap::heap()->reset_relocation_set();
  }

  // Phase 6: Concurrent Destroy Detached Pages
  {
    ZStatTimer timer(ZPhaseConcurrentDestroyDetachedPages);
    ZHeap::heap()->destroy_detached_pages();
  }

  // Phase 7: Concurrent Select Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentSelectRelocationSet);
    ZHeap::heap()->select_relocation_set();
  }

  // Phase 8: Concurrent Prepare Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentPrepareRelocationSet);
    ZHeap::heap()->prepare_relocation_set();
  }

  // Phase 9: Pause Relocate Start
  {
    ZRelocateStartClosure cl;
    vm_operation(&cl);
  }

  // Phase 10: Concurrent Relocate
  {
    ZStatTimer timer(ZPhaseConcurrentRelocated);
    ZHeap::heap()->relocate();
  }
}
```

以上10个阶段，只有Pause Mark Start（1、标记开始暂停）、Pause Mark End（3、标记结束暂停）和Pause Relocate Start（8、转移开始暂停）这三步中扫描GCRoots需要STW，其他过程都是并发的（G1的转移过程也是完全STW的，而且转移过程耗时和内存大小线性相关）。因为GCRoots数量并不多，所以能有效的控制停顿时间。

### ZGC默认参数

- ZGC基础参数
  ZGC参数初始化在这里`src\hotspot\share\gc\z\zArguments.cpp`
  ```cpp
  void ZArguments::initialize() {
    GCArguments::initialize();
  
    // Enable NUMA by default
    if (FLAG_IS_DEFAULT(UseNUMA)) {
      FLAG_SET_DEFAULT(UseNUMA, true);
    }
  
    // Disable biased locking by default
    if (FLAG_IS_DEFAULT(UseBiasedLocking)) {
      FLAG_SET_DEFAULT(UseBiasedLocking, false); //Biased优化任务在STW里执行，默认关闭这个优化
    }
  
    // Select number of parallel threads
    if (FLAG_IS_DEFAULT(ParallelGCThreads)) {
      FLAG_SET_DEFAULT(ParallelGCThreads, ZWorkers::calculate_nparallel()); //默认CPU核数的60%
    }
  
    if (ParallelGCThreads == 0) {
      vm_exit_during_initialization("The flag -XX:+UseZGC can not be combined with -XX:ParallelGCThreads=0");
    }
  
    // Select number of concurrent threads
    if (FLAG_IS_DEFAULT(ConcGCThreads)) {
      FLAG_SET_DEFAULT(ConcGCThreads, ZWorkers::calculate_nconcurrent()); //默认CPU核数的12.5%
    }
  
    if (ConcGCThreads == 0) {
      vm_exit_during_initialization("The flag -XX:+UseZGC can not be combined with -XX:ConcGCThreads=0");
    }
  #ifdef COMPILER2
    // Enable loop strip mining by default
    if (FLAG_IS_DEFAULT(UseCountedLoopSafepoints)) {
      FLAG_SET_DEFAULT(UseCountedLoopSafepoints, true);
      if (FLAG_IS_DEFAULT(LoopStripMiningIter)) {
        FLAG_SET_DEFAULT(LoopStripMiningIter, 1000);
      }
    }
  #endif
    // To avoid asserts in set_active_workers()
    FLAG_SET_DEFAULT(UseDynamicNumberOfGCThreads, true);
  
    // CompressedOops/UseCompressedClassPointers not supported
    FLAG_SET_DEFAULT(UseCompressedOops, false); // 不支持压缩指针
    FLAG_SET_DEFAULT(UseCompressedClassPointers, false); // 不支持压缩指针
  
    // ClassUnloading not (yet) supported
    FLAG_SET_DEFAULT(ClassUnloading, false);
    FLAG_SET_DEFAULT(ClassUnloadingWithConcurrentMark, false);
  
    // Verification before startup and after exit not (yet) supported
    FLAG_SET_DEFAULT(VerifyDuringStartup, false);
    FLAG_SET_DEFAULT(VerifyBeforeExit, false);
  
    // Verification before heap iteration not (yet) supported, for the
    // same reason we need fixup_partial_loads
    FLAG_SET_DEFAULT(VerifyBeforeIteration, false);
  
    // Verification of stacks not (yet) supported, for the same reason
    // we need fixup_partial_loads
    DEBUG_ONLY(FLAG_SET_DEFAULT(VerifyStack, false));
  }
  ```
- 高阶参数
  高阶参数在这里`src\hotspot\share\gc\z\z_globals.hpp`
  ```cpp
  #define GC_Z_FLAGS(develop,                                                 \
                     develop_pd,                                              \
                     product,                                                 \
                     product_pd,                                              \
                     diagnostic,                                              \
                     diagnostic_pd,                                           \
                     experimental,                                            \
                     notproduct,                                              \
                     manageable,                                              \
                     product_rw,                                              \
                     lp64_product,                                            \
                     range,                                                   \
                     constraint,                                              \
                     writeable)                                               \
                                                                              \
    product(ccstr, ZPath, NULL,                                               \
            "Filesystem path for Java heap backing storage "                  \
            "(must be a tmpfs or a hugetlbfs filesystem)")                    \
                                                                              \
    product(double, ZAllocationSpikeTolerance, 2.0,                           \
            "Allocation spike tolerance factor")                              \
                                                                              \
    product(double, ZFragmentationLimit, 25.0,                                \
            "Maximum allowed heap fragmentation")                             \
                                                                              \
    product(bool, ZStallOnOutOfMemory, true,                                  \
            "Allow Java threads to stall and wait for GC to complete "        \
            "instead of immediately throwing an OutOfMemoryError")            \
                                                                              \
    product(size_t, ZMarkStacksMax, NOT_LP64(512*M) LP64_ONLY(8*G),           \
            "Maximum number of bytes allocated for marking stacks")           \
            range(32*M, NOT_LP64(512*M) LP64_ONLY(1024*G))                    \
                                                                              \
    product(uint, ZCollectionInterval, 0,                                     \
            "Force GC at a fixed time interval (in seconds)")                 \
                                                                              \
    product(uint, ZStatisticsInterval, 10,                                    \
            "Time between statistics print outs (in seconds)")                \
            range(1, (uint)-1)                                                \
                                                                              \
    diagnostic(bool, ZStatisticsForceTrace, false,                            \
            "Force tracing of ZStats")                                        \
                                                                              \
    diagnostic(bool, ZProactive, true,                                        \
            "Enable proactive GC cycles")                                     \
                                                                              \
    diagnostic(bool, ZUnmapBadViews, false,                                   \
            "Unmap bad (inactive) heap views")                                \
                                                                              \
    diagnostic(bool, ZVerifyMarking, false,                                   \
            "Verify marking stacks")                                          \
                                                                              \
    diagnostic(bool, ZVerifyForwarding, false,                                \
            "Verify forwarding tables")                                       \
                                                                              \
    diagnostic(bool, ZSymbolTableUnloading, false,                            \
            "Unload unused VM symbols")                                       \
                                                                              \
    diagnostic(bool, ZWeakRoots, true,                                        \
            "Treat JNI WeakGlobalRefs and StringTable as weak roots")         \
                                                                              \
    diagnostic(bool, ZConcurrentStringTable, true,                            \
            "Clean StringTable concurrently")                                 \
                                                                              \
    diagnostic(bool, ZConcurrentVMWeakHandles, true,                          \
            "Clean VM WeakHandles concurrently")                              \
                                                                              \
    diagnostic(bool, ZConcurrentJNIWeakGlobalHandles, true,                   \
            "Clean JNI WeakGlobalRefs concurrently")                          \
                                                                              \
    diagnostic(bool, ZOptimizeLoadBarriers, true,                             \
            "Apply load barrier optimizations")                               \
                                                                              \
    develop(bool, ZVerifyLoadBarriers, false,                                 \
            "Verify that reference loads are followed by barriers")
  ```