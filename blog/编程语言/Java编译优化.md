### 前言
Java7引入了Tiered Compilation（分层编译）。
![Java-Tiered-Compilation.jpg](https://ping666.com/wp-content/uploads/2024/11/Java-Tiered-Compilation.jpg "Java-Tiered-Compilation.jpg")

javac编译期基本不做优化。绝大部分的编译优化是运行期JIT做的。C1的目标是编译速度，尽快的生成native code。C2的目标是代码效率，尽可能生成更快的native code。

Java10引入了Graal来代替C2。Graal是一款用Java写的编译器，有还更激进的优化策略。并且后来Graal独立发展，现在已经能将Java一步到位的编译成机器本地代码执行，就像C++做的那样。当然这和Java的某些高级特性是冲突的，比如反射。
>Build faster, smaller, leaner applications
An advanced JDK with ahead-of-time Native Image compilation

### inlining内联
- Trivial inlining小函数内联
- Call graph inlining通过函数调用图发现可以的内联
- Tail recursion elimination尾递归消除
```java
public static long fibonacciTailRec(long n, long a, long b) {
    if (n == 0) {
        return a;
    } else if (n == 1) {
        return b;
    } else {
        // 尾递归调用：这种模式的递归调用是有可能被优化掉的，使用一个基于循环的代码块代替
        return fibonacciTailRec(n - 1, b, a + b);
    }
}
```
- Virtual call guard optimizations识别出具体的类对象，直接替换以避免虚函数调用

内联相关的几个JVM参数：

- -XX:+Inline默认开启内联
- -XX:MaxInlineSize函数大小（字节数）超过这个值的不会内联，默认值35字节
- -XX:FreqInlineSize高频的函数字节数可以放宽到这个值，默认值325字节
- -XX:MaxInlineLevel最大内联嵌套深度，默认值15
- -XX:MaxRecursiveInlineLevel最大递归内联嵌套深度，默认值1
- -XX:InlineSmallCode编译后的函数大小超过这个值的不会内联，默认1000字节
- -XX:+UseInlineCaches默认开启内联缓存

### local optimizations局部优化
- Local data flow analyses and optimizations局部数据流分析优化
  - Constant Folding常量折叠
    比如：int x = 2 * 3.14；字节码中会直接放2 * 3.14的结果6.28；更进一步，如果x变量确定不会被修改，会直接换成常量。
  - Common Subexpression Elimination公共表达式消除
  - Dead Code Elimination消除不可达的代码
    比如：if (false) foo();
- Register usage optimization寄存器优化
- Simplifications of Java idioms常见Java用法优化

### control flow optimizations控制流优化
- Code reordering, splitting, and removal代码顺序重排/分割/去除
  现代的多核SMP，编译器都会做代码执行顺序重排，以并行计算来提升执行速度。
- Loop reduction and inversion循环减少和反转
- Loop striding and loop-invariant code motion循环步进/不变代码外提
- Loop unrolling and peeling循环展开/剥离
- Loop versioning and specialization循环版本化/专门化
- Exception-directed optimization异常处理优化
- Switch analysis

### global optimizations全局优化
代价昂贵的一些优化。

- Global data flow analyses and optimizations全局数据流分析与优化
- Partial redundancy elimination部分冗余消除
- Escape analysis逃逸分析
  当发现对象只在某个代码块局部使用时，会将这个局部对象标量替换后直接在栈上分配。栈上的对象会被自动销毁，降低GC压力。
- GC and memory allocation optimizations垃圾回收和内存分配优化
- Synchronization optimizations同步消除
  当发现一个使用了同步机制的对象，只会在一个线程中使用时，会直接消除同步操作。
