### 系统调用

![glibc.png](https://ping666.com/wp-content/uploads/2024/09/glibc.png "glibc.png")

上图描述了glibc的位置，在应用程序和内核之间。
一般的应用程序都是通过glibc调用内核接口的。比如`write -> sys_write -> __do_sys_write -> ksys_write -> vfs_write`中的sys_write就是内核接口。

内核提供的接口统一定义在`include\linux\syscalls.h`

### 核心结构

- 内核接口定义宏SYSCALL_DEFINE3
  下面的宏定义了`sys_write -> __do_sys_write -> ksys_write`这一段调用链。
```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	return ksys_write(fd, buf, count);
}
```

- 内核接口编号
  内核接口列表在sys_call_table数组上，每个接口都有唯一编号，定义在文件`include\uapi\asm-generic\unistd.h`，
```c
#define __NR_write 64
__SYSCALL(__NR_write, sys_write)
```

- pt_regs是系统调用时的寄存器状态
```c
struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
	unsigned long r15;
	unsigned long r14;
	unsigned long r13;
	unsigned long r12;
	unsigned long rbp;
	unsigned long rbx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
	unsigned long r11;
	unsigned long r10; //保存系统调用参数
	unsigned long r9; //保存系统调用参数
	unsigned long r8; //保存系统调用参数
	unsigned long rax; //保存系统调用号
	unsigned long rcx;
	unsigned long rdx; //保存系统调用参数
	unsigned long rsi; //保存系统调用参数
	unsigned long rdi; //保存系统调用参数
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
	unsigned long orig_rax;
/* Return frame for iretq */
	unsigned long rip;
	unsigned long cs;
	unsigned long eflags;
	unsigned long rsp;
	unsigned long ss;
/* top of stack page */
};
```

### 系统调用过程trap
32位的Linux内核是通过软中断0x80进行系统调用的，64位的通过机器指令syscall、sysret+寄存器MSR_LSTAR一起配合调用。

1. 保存用户态寄存器到pt_regs
1. 保存内核接口编号和参数到寄存器和pt_regs
1. 调用软中断0x80或者syscall进入内核态
1. 将运行级别从用户态3修改为内核态0
1. 通过接口编号和参数执行sys_call_table上对应的内核接口
1. 将运行级别从内核态0修改为用户态3
1. 软中段或者sysret返回
1. 从pt_regs恢复用户态寄存器状态

x86-32的glibc的实现在`sysdeps\unix\sysv\linux\i386\syscall.S`
```sass
	.text
ENTRY (syscall)

	PUSHARGS_6		/* Save register contents.  */
	_DOARGS_6(44)		/* Load arguments.  */
	movl 20(%esp), %eax	/* Load syscall number into %eax.  */
	ENTER_KERNEL		/* Do the system call.  */
	POPARGS_6		/* Restore register contents.  */
	cmpl $-4095, %eax	/* Check %eax for error.  */
	jae SYSCALL_ERROR_LABEL	/* Jump to error handler if error.  */
	ret			/* Return to caller.  */

PSEUDO_END (syscall)
```

![linux-trap-0x80-1.png](https://ping666.com/wp-content/uploads/2024/09/linux-trap-0x80-1.png "linux-trap-0x80-1.png")
