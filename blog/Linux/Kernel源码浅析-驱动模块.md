### 驱动简介
因为Linux是宏内核（Monolithic Kernel），所有的驱动程序作为内核的一部分一起运行。这大概也是普通程序员提交内核代码最简单的一个途径了。

### 内核模块
但是编译Linux内核是一件麻烦而且费时的事情，为了解决这个问题，内核提供了模块机制（Loadable Kernel Module(LKM)）。既能独立编译驱动程序，又能将其动态加载进内核。

- 模块代码入口
  内核模块接口在`include\linux\module.h`。可以看到每个模块其实是一个initcall。
```c
/* Each module must use one module_init(). */
#define module_init(initfn)					\
	static inline initcall_t __maybe_unused __inittest(void)		\
	{ return initfn; }					\
	int init_module(void) __copy(initfn)			\
		__attribute__((alias(#initfn)));		\
	___ADDRESSABLE(init_module, __initdata);
/* This is only required if you want to be unloadable. */
#define module_exit(exitfn)					\
	static inline exitcall_t __maybe_unused __exittest(void)		\
	{ return exitfn; }					\
	void cleanup_module(void) __copy(exitfn)		\
		__attribute__((alias(#exitfn)));		\
	___ADDRESSABLE(cleanup_module, __exitdata);
```

可以参考cdrom的代码`drivers\cdrom\cdrom.c`
```c
module_init(cdrom_init);
module_exit(cdrom_exit);
MODULE_DESCRIPTION("Uniform CD-ROM driver");
MODULE_LICENSE("GPL");

```
- 独立编译驱动模块
  编译的时候会自动生成一个module结构（设置好了init和exit等入口函数），编译最终结果是一个ELF格式的.ko文件。
  ELF是Linux使用的代码文件格式，常见的.o、.so、可执行程序都是这种格式，详情见[https://refspecs.linuxfoundation.org/elf/elf.pdf](https://refspecs.linuxfoundation.org/elf/elf.pdf "https://refspecs.linuxfoundation.org/elf/elf.pdf")

- 动态加载模块
  使用程序`/bin/kmod`加载模块会触发系统调用init_module或者finit_module，最终都会调用到load_module。代码在`kernel\module\main.c`。
  load_module函数里其实是做了可执行程序加载so类似的过程（link、relocate等）。

### 手写驱动

- 样例代码
  在Linux的源码目录./drivers下新建一个hello目录，hello.c代码如下
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
static int __init hello_init(void)
{
    printk(KERN_EMERG "[ KERN_EMERG ] Hello Module Init\n");
    printk( "[ default ] Hello Module Init\n");
    return 0;
}
static void __exit hello_exit(void)
{
    printk("[ default ]  Hello Module Exit\n");
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL2");
MODULE_AUTHOR("matthewliu");
MODULE_DESCRIPTION("hello module");
MODULE_ALIAS("test_module");
```

- Makefile
  Makefile代码如下
```bash
# SPDX-License-Identifier: GPL-2.0
obj-m += hello.o #不支持配置，如果要配置参见make menuconfig
```

### 编译驱动

- 开始编译
  在Linux源码跟目录下执行
```bash
cp /boot/config-6.6.34-9.oc9.x86_64 .
make //第一次要全编译一遍，不然编译模块会报Module.symvers is missing.
make M=./drivers/hello
```

- 编译依赖安装
  编译过程有什么依赖错误就安装对应的库
```bash
yum install openssl-devel //模块有签名校验
yum install elfutils-devel.x86_64 //模块是elf文件格式
```

### 安装驱动
记得用一个虚拟机什么的来尝试。。。
```bash
insmod hello.ko
rmmod hello.ko
```
dmesg查看模块打印的日志是否ok。
