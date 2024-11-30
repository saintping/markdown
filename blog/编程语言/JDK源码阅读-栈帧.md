### 线程栈
Linux机器上线程的栈（Stack）大小默认为8M。维护着这个线程里函数的调用关系，每个函数对应一帧（Frame）。

![stack-frame.png](https://ping666.com/wp-content/uploads/2024/09/stack-frame.png "stack-frame.png")

### x86_64栈帧
具体看下这段C程序在x86_64体系的CPU上栈帧是怎么维护的。

```c
int sum(int i, int j) {
    return i + j;
}
int main(int argc, char* args[]) {
    int x = sum(1, 2);
    return 0;
}
```

以上程序编译后，通过objdump -d导出汇编。main函数部分如下

```sass
000000000040050d <_Z3sumii>:
  40050d:	55                   	push   %rbp //把上一帧基址rbp压栈（注意这一步会改变rsp）
  40050e:	48 89 e5             	mov    %rsp,%rbp //把rbp指向上一帧的rsp（从这里可以看出保存上一个帧rbp的位置不算在这一帧里的）
  400511:	89 7d fc             	mov    %edi,-0x4(%rbp) //参数1
  400514:	89 75 f8             	mov    %esi,-0x8(%rbp) //参数2
  400517:	8b 45 f8             	mov    -0x8(%rbp),%eax //add参数寄存器
  40051a:	8b 55 fc             	mov    -0x4(%rbp),%edx //add参数寄存器
  40051d:	01 d0                	add    %edx,%eax //add结果在eax
  40051f:	5d                   	pop    %rbp //把rsp指向rbp（也就是销毁这一帧）
  400520:	c3                   	retq //把栈顶值压入rip（也就是执行return address）

0000000000400521 <main>:
  400521:	55                   	push   %rbp
  400522:	48 89 e5             	mov    %rsp,%rbp
  400525:	48 83 ec 20          	sub    $0x20,%rsp
  400529:	89 7d ec             	mov    %edi,-0x14(%rbp)
  40052c:	48 89 75 e0          	mov    %rsi,-0x20(%rbp)
  400530:	be 02 00 00 00       	mov    $0x2,%esi //参数2
  400535:	bf 01 00 00 00       	mov    $0x1,%edi //参数1
  40053a:	e8 ce ff ff ff       	callq  40050d <_Z3sumii> //push %rip(return address) + jump sum
  40053f:	89 45 fc             	mov    %eax,-0x4(%rbp)
  400542:	b8 00 00 00 00       	mov    $0x0,%eax
  400547:	c9                   	leaveq 
  400548:	c3                   	retq   
  400549:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
```

汇编指令callq、retq、push、pop和rbp（栈基地址寄存器）、rsp（栈顶地址寄存器）、rip（指令寄存器）一起配合构造起了程序的运行栈。
![gdb-stack.png](https://ping666.com/wp-content/uploads/2024/09/gdb-stack.png "gdb-stack.png")

x86-64指令集见[https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24592.pdf](https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24592.pdf "https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/24592.pdf")

插个题外话。
retq指令是完全信任并且去执行上一帧里保存的return地址的。这在正常情况下是没有问题的。
但是，栈是从高地址到低地址分配的，而帧上的局部变量是从低地址开始写的。如果局部变量因为内存越界，它其实覆盖的是高地址区域也就是上一帧的内容。在某种精心的设计下，可能就刚好把return地址改到了一个恶意的地址上。所以内存越界不只是简单的程序bug，更是严重的安全漏洞，是给黑客的特洛伊木马开了一道门。

### Java栈帧
上面x86_64指令集维护函数栈的过程，JVM中通过C++代码也做了类似的一个事情。

关于Java栈构造的代码在
`jdk\src\hotspot\share\runtime\javaCalls.cpp`

构造Java栈的最重要工作是JavaCallWrapper类完成的
```cpp
// A JavaCallWrapper is constructed before each JavaCall and destructed after the call.
// Its purpose is to allocate/deallocate a new handle block and to save/restore the last
// Java fp/sp. A pointer to the JavaCallWrapper is stored on the stack.

class JavaCallWrapper: StackObj {
  friend class VMStructs;
 private:
  JavaThread*      _thread;                 // the thread to which this call belongs
  JNIHandleBlock*  _handles;                // the saved handle block
  Method*          _callee_method;          // to be able to collect arguments if entry frame is top frame
  oop              _receiver;               // the receiver of the call (if a non-static call)

  JavaFrameAnchor  _anchor;                 // last thread anchor state that we must restore

  JavaValue*       _result;                 // result value
}
```

Java调用都是从JavaCalls::call_helper开始的。
`void JavaCalls::call(JavaValue* result, const methodHandle& method, JavaCallArguments* args, TRAPS)`
这个函数的工作

- 检查必要的上下文
- 检查method是否已编译，没有则编译
- 构造Java栈，JavaCallWrapper::JavaCallWrapper
  Java栈的地址和指令保存在JavaFrameAnchor对象上，JavaThread上有记录。
- 调用底层的Stub接口StubRoutines::call_stub执行method.entry_point
- 销毁Java栈，JavaCallWrapper::~JavaCallWrapper

构造Java栈的代码
```cpp
JavaCallWrapper::JavaCallWrapper(const methodHandle& callee_method, Handle receiver, JavaValue* result, TRAPS) {
  JavaThread* thread = (JavaThread *)THREAD;
  bool clear_pending_exception = true;

  guarantee(thread->is_Java_thread(), "crucial check - the VM thread cannot and must not escape to Java code");
  assert(!thread->owns_locks(), "must release all locks when leaving VM");
  guarantee(thread->can_call_java(), "cannot make java calls from the native compiler");
  _result   = result;

  // Allocate handle block for Java code. This must be done before we change thread_state to _thread_in_Java_or_stub,
  // since it can potentially block.
  JNIHandleBlock* new_handles = JNIHandleBlock::allocate_block(thread);

  // After this, we are official in JavaCode. This needs to be done before we change any of the thread local
  // info, since we cannot find oops before the new information is set up completely.
  ThreadStateTransition::transition(thread, _thread_in_vm, _thread_in_Java);

  // Make sure that we handle asynchronous stops and suspends _before_ we clear all thread state
  // in JavaCallWrapper::JavaCallWrapper(). This way, we can decide if we need to do any pd actions
  // to prepare for stop/suspend (flush register windows on sparcs, cache sp, or other state).
  if (thread->has_special_runtime_exit_condition()) {
    thread->handle_special_runtime_exit_condition();
    if (HAS_PENDING_EXCEPTION) {
      clear_pending_exception = false;
    }
  }


  // Make sure to set the oop's after the thread transition - since we can block there. No one is GC'ing
  // the JavaCallWrapper before the entry frame is on the stack.
  _callee_method = callee_method();
  _receiver = receiver();

#ifdef CHECK_UNHANDLED_OOPS
  THREAD->allow_unhandled_oop(&_receiver);
#endif // CHECK_UNHANDLED_OOPS

  _thread       = (JavaThread *)thread;
  _handles      = _thread->active_handles();    // save previous handle block & Java frame linkage

  // For the profiler, the last_Java_frame information in thread must always be in
  // legal state. We have no last Java frame if last_Java_sp == NULL so
  // the valid transition is to clear _last_Java_sp and then reset the rest of
  // the (platform specific) state.

  _anchor.copy(_thread->frame_anchor());
  _thread->frame_anchor()->clear();

  debug_only(_thread->inc_java_call_counter());
  _thread->set_active_handles(new_handles);     // install new handle block and reset Java frame linkage

  assert (_thread->thread_state() != _thread_in_native, "cannot set native pc to NULL");

  // clear any pending exception in thread (native calls start with no exception pending)
  if(clear_pending_exception) {
    _thread->clear_pending_exception();
  }

  if (_anchor.last_Java_sp() == NULL) {
    _thread->record_base_of_stack_pointer();
  }
}
```

销毁Java栈的代码
```cpp
JavaCallWrapper::~JavaCallWrapper() {
  assert(_thread == JavaThread::current(), "must still be the same thread");

  // restore previous handle block & Java frame linkage
  JNIHandleBlock *_old_handles = _thread->active_handles();
  _thread->set_active_handles(_handles);

  _thread->frame_anchor()->zap();

  debug_only(_thread->dec_java_call_counter());

  if (_anchor.last_Java_sp() == NULL) {
    _thread->set_base_of_stack_pointer(NULL);
  }


  // Old thread-local info. has been restored. We are not back in the VM.
  ThreadStateTransition::transition_from_java(_thread, _thread_in_vm);

  // State has been restored now make the anchor frame visible for the profiler.
  // Do this after the transition because this allows us to put an assert
  // the Java->vm transition which checks to see that stack is not walkable
  // on sparc/ia64 which will catch violations of the reseting of last_Java_frame
  // invariants (i.e. _flags always cleared on return to Java)

  _thread->frame_anchor()->copy(&_anchor);

  // Release handles after we are marked as being inside the VM again, since this
  // operation might block
  JNIHandleBlock::release_block(_old_handles, _thread);
}
```

最终的Java栈结构如下
![java-stack.png](https://ping666.com/wp-content/uploads/2024/09/java-stack.png "java-stack.png")

ZGC在初始标记阶段，扫描GC Roots对象就是从每个JavaThread线程的_anchor对象开始遍历的。
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