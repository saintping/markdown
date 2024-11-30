### 字节码基础知识
Java在前期都是解释执行的，虚拟机一行一行的执行字节码指令（这部分和CPython很像）。后面才陆续加入编译执行，即时编译JIT那些。

下面是一段典型的Java程序。
```java
package com.test.graphql;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GraphqlApplication {

    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(GraphqlApplication.class);
        springApplication.run(args);
    }

}
```
编译后，class文件里的main函数字节码是这样的

![jclasslib-opcode.png](https://ping666.com/wp-content/uploads/2024/09/jclasslib-opcode.png "jclasslib-opcode.png")

图中第1行new对应字节码187 (0xbb)，后面两个是参数。1，2，3行就是创建一个heap对象并且赋值给本地变量。13行invokespecial调用构造函数，19行调用虚函数，23行函数返回。
全部字节码已经有大概200个，在这个文件里
`src\jdk.compiler\share\classes\com\sun\tools\javac\jvm\Code.java`

完整的字节码规范可以看这里[https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html "https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html")

### 字节码解释执行
Hotspot虚拟机里为解释执行提供了两套Stub实现，CppInterpreter和TemplateInterpreter。前面是基于C++的，后面是基于汇编的。

虚拟机启动时会先初始化Stub代码块
void TemplateInterpreterGenerator::generate_all()
`src\hotspot\share\interpreter\templateInterpreterGenerator.cpp`
然后把Stub代码块和字节码对应起来
void TemplateTable::initialize()
`src\hotspot\share\interpreter\templateTable.cpp`
截取一小部分如下
```cpp
// Java spec bytecodes                ubcp|disp|clvm|iswd  in    out   generator             argument
  def(Bytecodes::_nop                 , ____|____|____|____, vtos, vtos, nop                 ,  _           );
  def(Bytecodes::_aconst_null         , ____|____|____|____, vtos, atos, aconst_null         ,  _           );
  def(Bytecodes::_iconst_m1           , ____|____|____|____, vtos, itos, iconst              , -1           );
  def(Bytecodes::_iconst_0            , ____|____|____|____, vtos, itos, iconst              ,  0           );
  def(Bytecodes::_iconst_1            , ____|____|____|____, vtos, itos, iconst              ,  1           );
  def(Bytecodes::_iconst_2            , ____|____|____|____, vtos, itos, iconst              ,  2           );
  def(Bytecodes::_iconst_3            , ____|____|____|____, vtos, itos, iconst              ,  3           );
  def(Bytecodes::_iconst_4            , ____|____|____|____, vtos, itos, iconst              ,  4           );
  def(Bytecodes::_iconst_5            , ____|____|____|____, vtos, itos, iconst              ,  5           );
  def(Bytecodes::_lconst_0            , ____|____|____|____, vtos, ltos, lconst              ,  0           );
  def(Bytecodes::_lconst_1            , ____|____|____|____, vtos, ltos, lconst              ,  1           );
  def(Bytecodes::_fconst_0            , ____|____|____|____, vtos, ftos, fconst              ,  0           );
  def(Bytecodes::_fconst_1            , ____|____|____|____, vtos, ftos, fconst              ,  1           );
  def(Bytecodes::_fconst_2            , ____|____|____|____, vtos, ftos, fconst              ,  2           );
  def(Bytecodes::_dconst_0            , ____|____|____|____, vtos, dtos, dconst              ,  0           );
  def(Bytecodes::_dconst_1            , ____|____|____|____, vtos, dtos, dconst              ,  1           );
  def(Bytecodes::_bipush              , ubcp|____|____|____, vtos, itos, bipush              ,  _           );
  def(Bytecodes::_sipush              , ubcp|____|____|____, vtos, itos, sipush              ,  _           );
  def(Bytecodes::_ldc                 , ubcp|____|clvm|____, vtos, vtos, ldc                 ,  false       );
```

运行时的解释执行基本就是：一个大switch，按字节码顺序跳表调用Stub代码，具体见下面函数。
BlockEnd* GraphBuilder::iterate_bytecodes_for_block(int bci)
`src\hotspot\share\c1\c1_GraphBuilder.cpp`

### 函数对象nmethod
Java类的方法最终被编译成nmethod对象
`jdk\src\hotspot\share\code\nmethod.hpp`

nmethod继承自CompiledMethod，而CompiledMethod的父类是CodeBlob。
```cpp
// nmethods (native methods) are the compiled code versions of Java methods.
//
// An nmethod contains:
//  - header                 (the nmethod structure)
//  [Relocation]
//  - relocation information
//  - constant part          (doubles, longs and floats used in nmethod)
//  - oop table
//  [Code]
//  - code body
//  - exception handler
//  - stub code
//  [Debugging information]
//  - oop array
//  - data array
//  - pcs
//  [Exception handler table]
//  - handler entry point array
//  [Implicit Null Pointer exception table]
//  - implicit null table array
class nmethod : public CompiledMethod {
    ...
}
```

可以看到nmethod上几乎没有关联的Java类和对象信息。它就是代码块和运行代码块的上下文。关联的Java对象是CodeBlob执行的一个普通参数。
在Java层是obj.method(args)，在虚拟机内部是method(obj, args)
