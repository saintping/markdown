### 什么是Safepoint
虚拟机在做垃圾回收前，都需要Stop The World。世界都停下来了就是进入了Safepoint。

Safepoint非常重要而又十分复杂，因为JVM的所有模块都和Safepoint相关，包括解释器、编译器、JNI、垃圾回收器。垃圾回收的一般策略是标记、整理、清除。这些操作都需要JVM中的其他线程全部停下来，处于一个静止的状态，垃圾回收器才能安全和有效的去扫描各种堆栈。

### 什么时候进入Safepoint
进入Safepoint，是一个多线程协调同步的过程。两个核心方法如下
```cpp
#src\hotspot\share\runtime\safepoint.hpp
SafepointSynchronize::begin()
SafepointSynchronize::end()
```

以下是不同的模块里，进入Safepoint的不同办法。
```cpp
1. Running interpreted
   The interpreter dispatch table is changed to force it to
   check for a safepoint condition between bytecodes.
2. Running in native code
   When returning from the native code, a Java thread must check
   the safepoint _state to see if we must block.  If the
   VM thread sees a Java thread in native, it does
   not wait for this thread to block.  The order of the memory
   writes and reads of both the safepoint state and the Java
   threads state is critical.  In order to guarantee that the
   memory writes are serialized with respect to each other,
   the VM thread issues a memory barrier instruction
   (on MP systems).  In order to avoid the overhead of issuing
   a memory barrier for each Java thread making native calls, each Java
   thread performs a write to a single memory page after changing
   the thread state.  The VM thread performs a sequence of
   mprotect OS calls which forces all previous writes from all
   Java threads to be serialized.  This is done in the
   os::serialize_thread_states() call.  This has proven to be
   much more efficient than executing a membar instruction
   on every call to native code.
3. Running compiled Code
   Compiled code reads a global (Safepoint Polling) page that
   is set to fault if we are trying to get to a safepoint.
4. Blocked
   A thread which is blocked will not be allowed to return from the
   block condition until the safepoint operation is complete.
5. In VM or Transitioning between states
   If a Java thread is currently running in the VM or transitioning
   between states, the safepointing code will wait for the thread to
   block itself when it attempts transitions to a new state.
```

### 怎么进入Safepoint
Hotspot采用JavaThread::_thread_state + polling page的方案来实现Safepoint。
Safepoint是一个高频操作，polling page更加轻量，这里没有用常见的内存屏障来同步。

```cpp
// JavaThreadState keeps track of which part of the code a thread is executing in. This
// information is needed by the safepoint code.
//
// There are 4 essential states:
//
//  _thread_new         : Just started, but not executed init. code yet (most likely still in OS init code)
//  _thread_in_native   : In native code. This is a safepoint region, since all oops will be in jobject handles
//  _thread_in_vm       : Executing in the vm
//  _thread_in_Java     : Executing either interpreted or compiled Java code (or could be in a stub)
//
// Each state has an associated xxxx_trans state, which is an intermediate state used when a thread is in
// a transition from one state to another. These extra states makes it possible for the safepoint code to
// handle certain thread_states without having to suspend the thread - making the safepoint code faster.
//
// Given a state, the xxxx_trans state can always be found by adding 1.
//
enum JavaThreadState {
  _thread_uninitialized     =  0, // should never happen (missing initialization)
  _thread_new               =  2, // just starting up, i.e., in process of being initialized
  _thread_new_trans         =  3, // corresponding transition state (not used, included for completness)
  _thread_in_native         =  4, // running in native code
  _thread_in_native_trans   =  5, // corresponding transition state
  _thread_in_vm             =  6, // running in VM
  _thread_in_vm_trans       =  7, // corresponding transition state
  _thread_in_Java           =  8, // running in Java or in stub code
  _thread_in_Java_trans     =  9, // corresponding transition state (not used, included for completness)
  _thread_blocked           = 10, // blocked in vm
  _thread_blocked_trans     = 11, // corresponding transition state
  _thread_max_state         = 12  // maximum thread state+1 - used for statistics allocation
};
```

简单点说就是线程保证在一定时期内会去轮询一下polling page，在需要进行GC时，线程会收到一个信号。当前线程在信号处理函数里生成该栈上的OopMaps，并且为马上到来的GC让路。
