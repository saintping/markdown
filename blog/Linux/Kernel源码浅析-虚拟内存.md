### 虚拟内存简介

上一篇进程调度的最后，在task_struct上下文切换那里有关于内存的处理，mm和active_mm。这里进一步学习一下内核的内存管理。

![linux-memory-model.jpg](https://ping666.com/wp-content/uploads/2024/09/linux-memory-model.jpg "linux-memory-model.jpg")

从上图可以看出，内核中的虚拟内存管理的核心是内存页。每页大小为4Kb，其他配置可以在`/proc/meminfo`查到。

### 内存核心结构
内存相关的结构定义都在`include\linux\mm_types.h`
```c
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/*
	 * Five words (20/40 bytes) are available in this union.
	 * WARNING: bit 0 of the first word is used for PageTail(). That
	 * means the other users of this union MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		struct {	/* Page cache and anonymous pages */
			/**
			 * @lru: Pageout list, eg. active_list protected by
			 * lruvec->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 */
			union {
				struct list_head lru;

				/* Or, for the Unevictable "LRU list" slot */
				struct {
					/* Always even, to negate PageTail */
					void *__filler;
					/* Count page's or folio's mlocks */
					unsigned int mlock_count;
				};

				/* Or, free page */
				struct list_head buddy_list;
				struct list_head pcp_list;
			};
			/* See page-flags.h for PAGE_MAPPING_FLAGS */
			struct address_space *mapping;
			union {
				pgoff_t index;		/* Our offset within mapping. */
				unsigned long share;	/* share count for fsdax */
			};
			/**
			 * @private: Mapping-private opaque data.
			 * Usually used for buffer_heads if PagePrivate.
			 * Used for swp_entry_t if PageSwapCache.
			 * Indicates order in the buddy system if PageBuddy.
			 */
			unsigned long private;
		};
		struct {	/* page_pool used by netstack */
			/**
			 * @pp_magic: magic value to avoid recycling non
			 * page_pool allocated pages.
			 */
			unsigned long pp_magic;
			struct page_pool *pp;
			unsigned long _pp_mapping_pad;
			unsigned long dma_addr;
			atomic_long_t pp_ref_count;
		};
		struct {	/* Tail pages of compound page */
			unsigned long compound_head;	/* Bit zero is set */
		};
		struct {	/* ZONE_DEVICE pages */
			/** @pgmap: Points to the hosting device page map. */
			struct dev_pagemap *pgmap;
			void *zone_device_data;
			/*
			 * ZONE_DEVICE private pages are counted as being
			 * mapped so the next 3 words hold the mapping, index,
			 * and private fields from the source anonymous or
			 * page cache page while the page is migrated to device
			 * private memory.
			 * ZONE_DEVICE MEMORY_DEVICE_FS_DAX pages also
			 * use the mapping, index, and private fields when
			 * pmem backed DAX files are mapped.
			 */
		};

		/** @rcu_head: You can use this to free a page by RCU. */
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		/*
		 * For head pages of typed folios, the value stored here
		 * allows for determining what this page is used for. The
		 * tail pages of typed folios will not store a type
		 * (page_type == _mapcount == -1).
		 *
		 * See page-flags.h for a list of page types which are currently
		 * stored here.
		 *
		 * Owners of typed folios may reuse the lower 16 bit of the
		 * head page page_type field after setting the page type,
		 * but must reset these 16 bit to -1 before clearing the
		 * page type.
		 */
		unsigned int page_type;

		/*
		 * For pages that are part of non-typed folios for which mappings
		 * are tracked via the RMAP, encodes the number of times this page
		 * is directly referenced by a page table.
		 *
		 * Note that the mapcount is always initialized to -1, so that
		 * transitions both from it and to it can be tracked, using
		 * atomic_inc_and_test() and atomic_add_negative(-1).
		 */
		atomic_t _mapcount;
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;

#ifdef CONFIG_MEMCG
	unsigned long memcg_data;
#elif defined(CONFIG_SLAB_OBJ_EXT)
	unsigned long _unused_slab_obj_exts;
#endif

	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif

#ifdef CONFIG_KMSAN
	/*
	 * KMSAN metadata for this page:
	 *  - shadow page: every bit indicates whether the corresponding
	 *    bit of the original page is initialized (0) or not (1);
	 *  - origin page: every 4 bytes contain an id of the stack trace
	 *    where the uninitialized value was created.
	 */
	struct page *kmsan_shadow;
	struct page *kmsan_origin;
#endif
} _struct_page_alignment;

struct maple_tree {
	union {
		spinlock_t	ma_lock;
		lockdep_map_p	ma_external_lock;
	};
	unsigned int	ma_flags;
	void __rcu      *ma_root; //B树
};

struct mm_struct {
	struct {
        ......
		struct maple_tree mm_mt; // 虚拟内存区域（VMA）

		unsigned long mmap_base;	/* base of mmap area */
		unsigned long mmap_legacy_base;	/* base of mmap area in bottom-up allocations */
        ......
		unsigned long task_size;	/* size of task vm space */
		pgd_t * pgd; //页全局目录
        ......
		struct list_head mmlist; /* List of maybe swapped mm's.	These
					  * are globally strung together off
					  * init_mm.mmlist, and are protected
					  * by mmlist_lock
					  */
        ......
		unsigned long hiwater_rss; /* High-watermark of RSS usage */
		unsigned long hiwater_vm;  /* High-water virtual memory usage */

		unsigned long total_vm;	   /* Total pages mapped */
		unsigned long locked_vm;   /* Pages that have PG_mlocked set */
		atomic64_t    pinned_vm;   /* Refcount permanently increased */
		unsigned long data_vm;	   /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
		unsigned long exec_vm;	   /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
		unsigned long stack_vm;	   /* VM_STACK */
		unsigned long def_flags;

		/**
		 * @write_protect_seq: Locked when any thread is write
		 * protecting pages mapped by this mm to enforce a later COW,
		 * for instance during page table copying for fork().
		 */
		seqcount_t write_protect_seq;

		spinlock_t arg_lock; /* protect the below fields */

		unsigned long start_code, end_code, start_data, end_data; //代码段和数据段
		unsigned long start_brk, brk, start_stack;
		unsigned long arg_start, arg_end, env_start, env_end;
        ......
		struct linux_binfmt *binfmt; //一般进程文件是elf格式

		/* Architecture-specific MM context */
		mm_context_t context; //上下文
-       ......
		/* store ref to file /proc/<pid>/exe symlink points to */
		struct file __rcu *exe_file; //进程文件
        ......
	} __randomize_layout;

	/*
	 * The mm_cpumask needs to be at the end of mm_struct, because it
	 * is dynamically sized based on nr_cpu_ids.
	 */
	unsigned long cpu_bitmap[];
};
```

### 内核态
回顾内核启动过程，内核一开始进入的是实模式（也就是程序直接访问物理内存），后面才切换到保护模式（虚拟内存模式，程序通过CPU提供的地址转换模块访问物理内存，起到保护物理内存的作用）。为什么需要先进入实模式呢？有两个原因：

- 引导程序很小，无法支持虚拟内存这一套复杂的逻辑
- 使用的内存很小（64kb），管理简单没必要使用更复杂的逻辑


内核在创建第一个用户进程systemd的时候，给kernel_clone传了CLONE_VM也就是共享虚拟内存。kernel_clone里处理内存mm_struct的代码在`kernel\fork.c`
```c
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
	struct mm_struct *mm, *oldmm;

	tsk->min_flt = tsk->maj_flt = 0;
	tsk->nvcsw = tsk->nivcsw = 0;
#ifdef CONFIG_DETECT_HUNG_TASK
	tsk->last_switch_count = tsk->nvcsw + tsk->nivcsw;
	tsk->last_switch_time = 0;
#endif

	tsk->mm = NULL; //tsk是新复制出来的PCB
	tsk->active_mm = NULL;

	/*
	 * Are we cloning a kernel thread?
	 *
	 * We need to steal a active VM for that..
	 */
	oldmm = current->mm; //current是当前PCB，如果是内核线程mm不空
	if (!oldmm)
		return 0;

	if (clone_flags & CLONE_VM) {
		mmget(oldmm); //引用计数mm_users+1
		mm = oldmm;
	} else {
		mm = dup_mm(tsk, current->mm); //新建一个mm_stask，并且复制exe_file
		if (!mm)
			return -ENOMEM;
	}

	tsk->mm = mm;
	tsk->active_mm = mm;
	sched_mm_cid_fork(tsk);
	return 0;
}
```
创建用户进程时有CLONE_VM标记，也就是每个用户进程都继承了内核的虚拟地址空间（内核态）。这个地址空间，大部分的用户进程其实很少修改或者只修改其中很小的一部分。所以内核在这里使用了RCU技术（Read-Copy Update），只有在涉及修改的时候才真正复制一份新的出来使用。内核中很多数据结构都是支持RCU的。

内核加载的时候，地址空间是从低位地址0开始的。但是在用户进程里却是在高地址空间（32位机器是3G~4G），用户自己的内存是从低地址（32位机器是0G~3G）开始的。这其实是内存寻址时对内核态空间做了保护，在地址前面统一加了标记。

### 硬件寻址支持
x86架构为Linux的虚拟内存管理提供了两大支持：

1. CR3（Control Register 3）
   CR3寄存器用来保存当前内存页目录基址。
2. MMU（Memory Management Unit，内存管理单元）
   - 将虚拟地址转换成物理地址
   - 内存地址保护
     程序运行中常见的Segment Fault，就是MMU检测并触发的。
   - TLB（Translation Lookaside Buffer）内存页缓存
   - 设备内存映射

下面是虚拟内存到物理内存的映射关系（4级页目录）。
[![linux-memory-pgd.png](https://ping666.com/wp-content/uploads/2024/09/linux-memory-pgd.png "linux-memory-pgd.png")](https://ping666.com/wp-content/uploads/2024/09/linux-memory-pgd.png "linux-memory-pgd.png")

更详细的内存映射关系可以看这个[https://www.cnblogs.com/binlovetech/p/17571929.html](https://www.cnblogs.com/binlovetech/p/17571929.html "https://www.cnblogs.com/binlovetech/p/17571929.html")

### 内核内存分配
kmalloc的代码在`include\linux\slab.h`
```c
static __always_inline __alloc_size(1) void *kmalloc_noprof(size_t size, gfp_t flags)
{
	if (__builtin_constant_p(size) && size) { //预分配的一些对象池，大小从32字节到2M都有
		unsigned int index;

		if (size > KMALLOC_MAX_CACHE_SIZE)
			return __kmalloc_large_noprof(size, flags);

		index = kmalloc_index(size);
		return __kmalloc_cache_noprof(
				kmalloc_caches[kmalloc_type(flags, _RET_IP_)][index],
				flags, size);
	}
	return __kmalloc_noprof(size, flags); //从slub中分配
}
#define kmalloc(...)				alloc_hooks(kmalloc_noprof(__VA_ARGS__)) //alloc_hooks只是为了打标codetag（__FILE__/__LINE__/__FUNC__这些）
```

继续看__kmalloc_noprof，代码在`mm\slub.c`
```c
void *__kmalloc_noprof(size_t size, gfp_t flags)
{
	return __do_kmalloc_node(size, NULL, flags, NUMA_NO_NODE, _RET_IP_);
}

static __always_inline
void *__do_kmalloc_node(size_t size, kmem_buckets *b, gfp_t flags, int node,
			unsigned long caller)
{
	struct kmem_cache *s;
	void *ret;

	if (unlikely(size > KMALLOC_MAX_CACHE_SIZE)) { //大于一页4kb的内存
		ret = __kmalloc_large_node_noprof(size, flags, node);
		trace_kmalloc(caller, ret, size,
			      PAGE_SIZE << get_order(size), flags, node);
		return ret;
	}

	if (unlikely(!size))
		return ZERO_SIZE_PTR;

	s = kmalloc_slab(size, b, flags, caller); //分配slab

	ret = slab_alloc_node(s, NULL, flags, node, caller, size); //分配node
	ret = kasan_kmalloc(s, ret, size, flags); //标记内存
	trace_kmalloc(caller, ret, size, s->size, flags, node);
	return ret;
}
```

分配内存主要完成两件事，一个是标记物理内存已使用，一个是维护4级页目录映射关系。会生成两个重要结构vmap_area和vm_struct
```c
struct vm_struct {
	struct vm_struct	*next;
	void			*addr; //内存首地址
	unsigned long		size; //内存大小
	unsigned long		flags;
	struct page		**pages; //内存页
#ifdef CONFIG_HAVE_ARCH_HUGE_VMALLOC
	unsigned int		page_order;
#endif
	unsigned int		nr_pages;
	phys_addr_t		phys_addr;
	const void		*caller;
};

struct vmap_area {
	unsigned long va_start; //分配内存的起始地址
	unsigned long va_end; //分配内存的结束地址

	struct rb_node rb_node;         /* address sorted rbtree */
	struct list_head list;          /* address sorted list */

	/*
	 * The following two variables can be packed, because
	 * a vmap_area object can be either:
	 *    1) in "free" tree (root is free_vmap_area_root)
	 *    2) or "busy" tree (root is vmap_area_root)
	 */
	union {
		unsigned long subtree_max_size; /* in "free" tree */
		struct vm_struct *vm;           /* in "busy" tree */
	};
	unsigned long flags; /* mark type of vm_map_ram area */
};
```
