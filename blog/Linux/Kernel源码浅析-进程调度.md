### 什么是进程
上一篇内核启动的最后提到，rest_init函数里开启了1号进程systemd。

现代操作系统都是多任务的，支持多CPU并行调度。每个任务对应一个进程，可以看成是一组资源集合的描述：程序代码块、内存栈、打开的文件、调度的资源等。
![linux-process-model.png](https://ping666.com/wp-content/uploads/2024/09/linux-process-model.png "linux-process-model.png")

Linux内核中，一个进程就是一个task_struct（也叫PCB，进程控制块）。task_struct结构定义在`include\linux\sched.h`。总共有300+字段，省略一些不重要的之后如下：
```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
  /*
   * For reasons of header soup (see current_thread_info()), this
   * must be the first element of task_struct.
   */
  struct thread_info        thread_info; //保存和架构相关的信息，不同架构下thread_info的定义不同
#endif
	unsigned int			__state; //进程运行状态
    ......
	void				*stack; //进程栈
    ......
	int				prio; // 调度优先级
	int				static_prio;
	int				normal_prio;
	unsigned int			rt_priority;
    const struct sched_class	*sched_class; //调度类
    ......
	unsigned int			policy; //调度策略
    ......
	struct list_head		tasks; //当前所在任务队列
    ......
	struct mm_struct		*mm; //进程的用户内存空间，内核线程为null
	struct mm_struct		*active_mm;
	struct address_space		*faults_disabled_mapping;
	int				exit_state; //进程退出相关的标记
	int				exit_code;
	int				exit_signal;
    ......
	pid_t				pid; //进程id
	pid_t				tgid;
    ......
	/* Recipient of SIGCHLD, wait4() reports: */
	struct task_struct __rcu	*parent; //父进程
	/*
	 * Children/sibling form the list of natural children:
	 */
	struct list_head		children; //子进程
	struct list_head		sibling; //兄弟进程
	struct task_struct		*group_leader; //进程组
    ......
	u64				utime; //执行时间
	u64				stime;
    ......
	/* Filesystem information: */
	struct fs_struct		*fs; //文件系统
	/* Open file information: */
	struct files_struct		*files; //打开的文件列表
    ......
	/* Signal handlers: */
	struct signal_struct		*signal; //信号
	struct sighand_struct __rcu		*sighand;
    ......
};
```

内核代码中通过current宏可以很方便拿到当前进程结构
```c
struct pcpu_hot {
	union {
		struct {
			struct task_struct	*current_task;
			int			preempt_count;
			int			cpu_number;
#ifdef CONFIG_MITIGATION_CALL_DEPTH_TRACKING
			u64			call_depth;
#endif
			unsigned long		top_of_stack;
			void			*hardirq_stack_ptr;
			u16			softirq_pending;
#ifdef CONFIG_X86_64
			bool			hardirq_stack_inuse;
#else
			void			*softirq_stack_ptr;
#endif
		};
		u8	pad[64];
	};
};
static_assert(sizeof(struct pcpu_hot) == 64);

DECLARE_PER_CPU_ALIGNED(struct pcpu_hot, pcpu_hot);

/* const-qualified alias to pcpu_hot, aliased by linker. */
DECLARE_PER_CPU_ALIGNED(const struct pcpu_hot __percpu_seg_override,
			const_pcpu_hot);

static __always_inline struct task_struct *get_current(void)
{
	if (IS_ENABLED(CONFIG_USE_X86_SEG_SUPPORT))
		return this_cpu_read_const(const_pcpu_hot.current_task);

	return this_cpu_read_stable(pcpu_hot.current_task);
}

#define current get_current()
```

### 进程创建
内核中创建进程统一通过kernel_clone函数，kernel_clone代码在`kernel\fork.c`。
kernel_clone的核心功能就是复制PCB（dup_task_struct），按标记复制各种资源，并且开始调度（activate_task）。

- dup_task_struct
  创建一个新的task_struct，然后是各种复制操作。
- 复制资源
  用户态进程一般标记都有CLONE_VM、CLONE_FS，复制虚拟内存空间和文件系统
```c
/*
 * cloning flags:
 */
#define CSIGNAL		0x000000ff	/* signal mask to be sent at exit */
#define CLONE_VM	0x00000100	/* set if VM shared between processes */
#define CLONE_FS	0x00000200	/* set if fs info shared between processes */
#define CLONE_FILES	0x00000400	/* set if open files shared between processes */
#define CLONE_SIGHAND	0x00000800	/* set if signal handlers and blocked signals shared */
#define CLONE_PIDFD	0x00001000	/* set if a pidfd should be placed in parent */
#define CLONE_PTRACE	0x00002000	/* set if we want to let tracing continue on the child too */
#define CLONE_VFORK	0x00004000	/* set if the parent wants the child to wake it up on mm_release */
#define CLONE_PARENT	0x00008000	/* set if we want to have the same parent as the cloner */
#define CLONE_THREAD	0x00010000	/* Same thread group? */
#define CLONE_NEWNS	0x00020000	/* New mount namespace group */
#define CLONE_SYSVSEM	0x00040000	/* share system V SEM_UNDO semantics */
#define CLONE_SETTLS	0x00080000	/* create a new TLS for the child */
#define CLONE_PARENT_SETTID	0x00100000	/* set the TID in the parent */
#define CLONE_CHILD_CLEARTID	0x00200000	/* clear the TID in the child */
#define CLONE_DETACHED		0x00400000	/* Unused, ignored */
#define CLONE_UNTRACED		0x00800000	/* set if the tracing process can't force CLONE_PTRACE on this clone */
#define CLONE_CHILD_SETTID	0x01000000	/* set the TID in the child */
#define CLONE_NEWCGROUP		0x02000000	/* New cgroup namespace */
#define CLONE_NEWUTS		0x04000000	/* New utsname namespace */
#define CLONE_NEWIPC		0x08000000	/* New ipc namespace */
#define CLONE_NEWUSER		0x10000000	/* New user namespace */
#define CLONE_NEWPID		0x20000000	/* New pid namespace */
#define CLONE_NEWNET		0x40000000	/* New network namespace */
#define CLONE_IO		0x80000000	/* Clone io context */
```

- activate_task
  将新的task_struct放进调度队列
```c
void activate_task(struct rq *rq, struct task_struct *p, int flags)
{
	if (task_on_rq_migrating(p))
		flags |= ENQUEUE_MIGRATED;
	if (flags & ENQUEUE_MIGRATED)
		sched_mm_cid_migrate_to(rq, p);

	enqueue_task(rq, p, flags);

	WRITE_ONCE(p->on_rq, TASK_ON_RQ_QUEUED);
	ASSERT_EXCLUSIVE_WRITER(p->on_rq);
}
```

### 轻量级进程
应用为了程序的提高并发度，一般会在进程内部开启多线程。线程会共享进程的大部分资源（代码块、内存、文件描述符、信号等），只是分开调度。

线程调度和进程调度是两个东西。但是Linux内核为了简化调度模块，将线程看成是轻量级进程LWP（Light Weight Process）。LWP也有PCB结构，和普通进程共用一个调度，只是需要复制和装载的东西很少。

### 进程调度
Linux进程的状态图如下：
![linux-process-state.png](https://ping666.com/wp-content/uploads/2024/11/linux-process-state.png "linux-process-state.png")

调度的代码在`kernel\sched\core.c`。
最开始（比如0.11版本）的调度很简单：从队列rq中取出task_struct交给处理器执行，循环执行就可以了。虽然每一遍都需要精巧的处理上下文，整体还是比较好理解的。
后来为了支持SMP（Symmetrical Multi-Processing），调度的复杂度就有点变态的了。囫囵吞枣式的看了一下，详情代码
```c
static void __sched notrace __schedule(unsigned int sched_mode)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	unsigned long prev_state;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu); //取当前处理器的队列，每个处理器自己的任务队列
	prev = rq->curr;

	schedule_debug(prev, !!sched_mode);

	if (sched_feat(HRTICK) || sched_feat(HRTICK_DL))
		hrtick_clear(rq);

	local_irq_disable(); //暂停当前中断
	rcu_note_context_switch(!!sched_mode);

	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);
	rq->clock_update_flags = RQCF_UPDATED;

	switch_count = &prev->nivcsw;

	prev_state = READ_ONCE(prev->__state);
	if (!(sched_mode & SM_MASK_PREEMPT) && prev_state) { //是否抢占式调度
		if (signal_pending_state(prev_state, prev)) {
			WRITE_ONCE(prev->__state, TASK_RUNNING);
		} else {
			prev->sched_contributes_to_load =
				(prev_state & TASK_UNINTERRUPTIBLE) &&
				!(prev_state & TASK_NOLOAD) &&
				!(prev_state & TASK_FROZEN);

			if (prev->sched_contributes_to_load)
				rq->nr_uninterruptible++;

			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}
		}
		switch_count = &prev->nvcsw;
	}

	next = pick_next_task(rq, prev, &rf);
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();
#ifdef CONFIG_SCHED_DEBUG
	rq->last_seen_need_resched_ns = 0;
#endif

	if (likely(prev != next)) {
		rq->nr_switches++;

		RCU_INIT_POINTER(rq->curr, next); //PCB是Red-Copy Update方式

		++*switch_count;

		migrate_disable_switch(rq, prev);
		psi_account_irqtime(rq, prev, next);
		psi_sched_switch(prev, next, !task_on_rq_queued(prev));

		trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next, prev_state);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf); //切换到新任务的上下文（寄存器、内存等）
	} else {
		rq_unpin_lock(rq, &rf);
		__balance_callbacks(rq); //rq队列没有任务，开始多核负载均衡
		raw_spin_rq_unlock_irq(rq);
	}
}
```

### 进程切换
context_switch是将上下文切换到新进程的方法。将运行时的上下文（比如内存地址空间，寄存器状态）都切换成新进程的状态，CPU处理器下次从RIP（Register Instruction Pointer，寄存器指令指针）开始执行时，就已经是在运行新进程了。
```c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
	       struct task_struct *next, struct rq_flags *rf)
{
	prepare_task_switch(rq, prev, next);

	/*
	 * For paravirt, this is coupled with an exit in switch_to to
	 * combine the page table reload and the switch backend into
	 * one hypercall.
	 */
	arch_start_context_switch(prev);

	/*
	 * kernel -> kernel   lazy + transfer active
	 *   user -> kernel   lazy + mmgrab_lazy_tlb() active
	 *
	 * kernel ->   user   switch + mmdrop_lazy_tlb() active
	 *   user ->   user   switch
	 *
	 * switch_mm_cid() needs to be updated if the barriers provided
	 * by context_switch() are modified.
	 */
	if (!next->mm) {                                // to kernel
		enter_lazy_tlb(prev->active_mm, next);

		next->active_mm = prev->active_mm;
		if (prev->mm)                           // from user
			mmgrab_lazy_tlb(prev->active_mm);
		else
			prev->active_mm = NULL;
	} else {                                        // to user
		membarrier_switch_mm(rq, prev->active_mm, next->mm);
		/*
		 * sys_membarrier() requires an smp_mb() between setting
		 * rq->curr / membarrier_switch_mm() and returning to userspace.
		 *
		 * The below provides this either through switch_mm(), or in
		 * case 'prev->active_mm == next->mm' through
		 * finish_task_switch()'s mmdrop().
		 */
		switch_mm_irqs_off(prev->active_mm, next->mm, next);
		lru_gen_use_mm(next->mm);

		if (!prev->mm) {                        // from kernel
			/* will mmdrop_lazy_tlb() in finish_task_switch(). */
			rq->prev_mm = prev->active_mm;
			prev->active_mm = NULL;
		}
	}

	/* switch_mm_cid() requires the memory barriers above. */
	switch_mm_cid(rq, prev, next);

	prepare_lock_switch(rq, next, rf);

	/* Here we just switch the register state and the stack. */
	switch_to(prev, next, prev);
	barrier();

	return finish_task_switch(prev);
}
```

### 调度策略
内核提供了多种调度策略类：stop_sched_class、dl_sched_class、rt_sched_class、fair_sched_class、idle_sched_class。
现在一般使用公平调度策略，fair_sched_class的方法列表如下
```c
DEFINE_SCHED_CLASS(fair) = {

	.enqueue_task		= enqueue_task_fair,
	.dequeue_task		= dequeue_task_fair,
	.yield_task		= yield_task_fair,
	.yield_to_task		= yield_to_task_fair,

	.wakeup_preempt		= check_preempt_wakeup_fair,

	.pick_next_task		= __pick_next_task_fair,
	.put_prev_task		= put_prev_task_fair,
	.set_next_task          = set_next_task_fair,

#ifdef CONFIG_SMP
	.balance		= balance_fair,
	.pick_task		= pick_task_fair,
	.select_task_rq		= select_task_rq_fair,
	.migrate_task_rq	= migrate_task_rq_fair,

	.rq_online		= rq_online_fair,
	.rq_offline		= rq_offline_fair,

	.task_dead		= task_dead_fair,
	.set_cpus_allowed	= set_cpus_allowed_fair,
#endif

	.task_tick		= task_tick_fair,
	.task_fork		= task_fork_fair,

	.prio_changed		= prio_changed_fair,
	.switched_from		= switched_from_fair,
	.switched_to		= switched_to_fair,

	.get_rr_interval	= get_rr_interval_fair,

	.update_curr		= update_curr_fair,

#ifdef CONFIG_FAIR_GROUP_SCHED
	.task_change_group	= task_change_group_fair,
#endif

#ifdef CONFIG_SCHED_CORE
	.task_is_throttled	= task_is_throttled_fair,
#endif

#ifdef CONFIG_UCLAMP_TASK
	.uclamp_enabled		= 1,
#endif
};
```
