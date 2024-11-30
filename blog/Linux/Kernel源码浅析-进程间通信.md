### 前言
因为进程的隔离，进程之间是无法直接访问对方的，需要通过内核的帮助来传递信息。

### 信号signal
如果只是传递简单的控制信息，signal是一个高效的选择。

- 信号相关的PCB字段
```c
struct task_struct {
    ......
	/* Signal handlers: */
	struct signal_struct		*signal;
	struct sighand_struct __rcu		*sighand; //信号和处理函数映射表
	sigset_t			blocked; //拒绝接收列表（SIGKILL除外）
	sigset_t			real_blocked;
	/* Restored if set_restore_sigmask() was used: */
	sigset_t			saved_sigmask;
	struct sigpending		pending; //未处理的信号列表：0~31的信号重复会直接丢弃（兼容UNIX方案），32~63的信号直接入队。
    ......
}
typedef struct {
	unsigned long sig[_NSIG_WORDS]; //64bits，32位机器上_NSIG_WORDS=2
} sigset_t;
struct sigpending {
	struct list_head list;
	sigset_t signal;
};
struct sighand_struct {
	spinlock_t		siglock;
	refcount_t		count;
	wait_queue_head_t	signalfd_wqh;
	struct k_sigaction	action[_NSIG];
};
struct k_sigaction {
	struct sigaction sa;
#ifdef __ARCH_HAS_KA_RESTORER
	__sigrestore_t ka_restorer;
#endif
};
struct sigaction {
#ifndef __ARCH_HAS_IRIX_SIGACTION
	__sighandler_t	sa_handler;
	unsigned long	sa_flags;
#else
	unsigned int	sa_flags;
	__sighandler_t	sa_handler;
#endif
#ifdef __ARCH_HAS_SA_RESTORER
	__sigrestore_t sa_restorer;
#endif
	sigset_t	sa_mask;	/* mask last for extensibility */
};
typedef void __signalfn_t(int); //信号处理函数的定义
typedef __signalfn_t __user *__sighandler_t;
```

- 设置信号处理函数
  1. glibc的sigaction
  1. 系统调用rt_sigaction
```c
SYSCALL_DEFINE4(rt_sigaction, int, sig,
		const struct sigaction __user *, act,
		struct sigaction __user *, oact,
		size_t, sigsetsize)
{
	struct k_sigaction new_sa, old_sa;
	int ret;

	/* XXX: Don't preclude handling different sized sigset_t's.  */
	if (sigsetsize != sizeof(sigset_t))
		return -EINVAL;

	if (act && copy_from_user(&new_sa.sa, act, sizeof(new_sa.sa)))
		return -EFAULT;

	ret = do_sigaction(sig, act ? &new_sa : NULL, oact ? &old_sa : NULL); //设置到PCB上
	if (ret)
		return ret;

	if (oact && copy_to_user(oact, &old_sa.sa, sizeof(old_sa.sa)))
		return -EFAULT;

	return 0;
}  
```

- 发送信号
  1. glibc函数kill
  1. 系统调用kill
```c
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct kernel_siginfo info;

	prepare_kill_siginfo(sig, &info, PIDTYPE_TGID);

	return kill_something_info(sig, &info, pid);
}
static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
	int ret;

	if (pid > 0) //>0发送给正常的pid进程
		return kill_proc_info(sig, info, pid);

	/* -INT_MIN is undefined.  Exclude this case to avoid a UBSAN warning */
	if (pid == INT_MIN)
		return -ESRCH;

	read_lock(&tasklist_lock);
	if (pid != -1) {
		ret = __kill_pgrp_info(sig, info,
				pid ? find_vpid(-pid) : task_pgrp(current)); //0发送给当前进程组下所有进程，<-1发送给当前进程组的-pid进程
	} else { //-1发送给当前用户下的所有进程
		int retval = 0, count = 0;
		struct task_struct * p;

		for_each_process(p) {
			if (task_pid_vnr(p) > 1 &&
					!same_thread_group(p, current)) {
				int err = group_send_sig_info(sig, info, p,
							      PIDTYPE_MAX);
				++count;
				if (err != -EPERM)
					retval = err;
			}
		}
		ret = count ? retval : -ESRCH;
	}
	read_unlock(&tasklist_lock);

	return ret;
}
```

### 管道pipe
最常见的是匿名管道，比如
`cat test.txt | grep 'hello world'`
cat进程打开匿名管道，关闭读端，把内容从写端写入。cat进程fork出新进程grep，新进程关闭写端，从读端读出内容。
这种匿名管道只能在有亲缘关系的进程间使用。没有亲缘关系的进程只能使用命名管道，管道和一个文件绑定。

内核中pipe对应的pipefs，一个内存文件系统。代码在`include\linux\pipe_fs_i.h`

- 创建管道
  1. glibc函数pipe
  1. 系统调用pipe
```c
SYSCALL_DEFINE1(pipe, int __user *, fildes)
{
	return do_pipe2(fildes, 0);
}
static int do_pipe2(int __user *fildes, int flags)
{
	struct file *files[2];
	int fd[2];
	int error;

	error = __do_pipe_flags(fd, files, flags);
	if (!error) {
		if (unlikely(copy_to_user(fildes, fd, sizeof(fd)))) {
			fput(files[0]);
			fput(files[1]);
			put_unused_fd(fd[0]);
			put_unused_fd(fd[1]);
			error = -EFAULT;
		} else {
			fd_install(fd[0], files[0]);
			fd_install(fd[1], files[1]);
		}
	}
	return error;
}
static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
	int error;
	int fdw, fdr;

	if (flags & ~(O_CLOEXEC | O_NONBLOCK | O_DIRECT | O_NOTIFICATION_PIPE))
		return -EINVAL;

	error = create_pipe_files(files, flags); //在pipefs文件系统上次创建对应的inode
	if (error)
		return error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_read_pipe;
	fdr = error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_fdr;
	fdw = error;

	audit_fd_pair(fdr, fdw);
	fd[0] = fdr;
	fd[1] = fdw;
	/* pipe groks IOCB_NOWAIT */
	files[0]->f_mode |= FMODE_NOWAIT;
	files[1]->f_mode |= FMODE_NOWAIT;
	return 0;
 err_fdr:
	put_unused_fd(fdr);
 err_read_pipe:
	fput(files[0]);
	fput(files[1]);
	return error;
}
```

- 管道读写
  管道就是pipefs文件系统上的inode，通过文件接口读写。

### 信号量semaphores
内核中的信号量结构
```c
/* One semaphore structure for each semaphore in the system. */
struct sem {
	int	semval;		/* current value */
	/*
	 * PID of the process that last modified the semaphore. For
	 * Linux, specifically these are:
	 *  - semop
	 *  - semctl, via SETVAL and SETALL.
	 *  - at task exit when performing undo adjustments (see exit_sem).
	 */
	struct pid *sempid;
	spinlock_t	lock;	/* spinlock for fine-grained semtimedop */
	struct list_head pending_alter; /* pending single-sop operations */
					/* that alter the semaphore */
	struct list_head pending_const; /* pending single-sop operations */
					/* that do not alter the semaphore*/
	time64_t	 sem_otime;	/* candidate for sem_otime */
} ____cacheline_aligned_in_smp;
```
经典的互斥量lock + 信号值semval，配合up和down操作。代码在`ipc\sem.c`

### 消息队列message queues
消息队列在内核中是一个list。
```c
/* one msq_queue structure for each present queue on the system */
struct msg_queue {
	struct kern_ipc_perm q_perm; //ipc权限相关
	time64_t q_stime;		/* last msgsnd time */
	time64_t q_rtime;		/* last msgrcv time */
	time64_t q_ctime;		/* last change time */
	unsigned long q_cbytes;		/* current number of bytes on queue */
	unsigned long q_qnum;		/* number of messages in queue */
	unsigned long q_qbytes;		/* max number of bytes on queue */
	struct pid *q_lspid;		/* pid of last msgsnd */
	struct pid *q_lrpid;		/* last receive pid */

	struct list_head q_messages; //消息列表
	struct list_head q_receivers; //阻塞的接收进程列表
	struct list_head q_senders; //阻塞的发送进程列表
} __randomize_layout;
```

- 创建消息队列msgget
```c
SYSCALL_DEFINE2(msgget, key_t, key, int, msgflg)
{
	return ksys_msgget(key, msgflg);
}
long ksys_msgget(key_t key, int msgflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops msg_ops = {
		.getnew = newque,
		.associate = security_msg_queue_associate,
	};
	struct ipc_params msg_params;

	ns = current->nsproxy->ipc_ns;

	msg_params.key = key;
	msg_params.flg = msgflg;

	return ipcget(ns, &msg_ids(ns), &msg_ops, &msg_params);
}
int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
			const struct ipc_ops *ops, struct ipc_params *params)
{
	if (params->key == IPC_PRIVATE)
		return ipcget_new(ns, ids, ops, params);
	else
		return ipcget_public(ns, ids, ops, params);
}
static int ipcget_new(struct ipc_namespace *ns, struct ipc_ids *ids,
		const struct ipc_ops *ops, struct ipc_params *params)
{
	int err;

	down_write(&ids->rwsem);
	err = ops->getnew(ns, params);
	up_write(&ids->rwsem);
	return err;
}
```
- 发送消息msgsnd
```c
SYSCALL_DEFINE4(msgsnd, int, msqid, struct msgbuf __user *, msgp, size_t, msgsz,
		int, msgflg)
{
	return ksys_msgsnd(msqid, msgp, msgsz, msgflg);
}
long ksys_msgsnd(int msqid, struct msgbuf __user *msgp, size_t msgsz,
		 int msgflg)
{
	long mtype;

	if (get_user(mtype, &msgp->mtype))
		return -EFAULT;
	return do_msgsnd(msqid, mtype, msgp->mtext, msgsz, msgflg);
}
static long do_msgsnd(int msqid, long mtype, void __user *mtext,
		size_t msgsz, int msgflg)
{
	struct msg_queue *msq;
	struct msg_msg *msg;
	int err;
	struct ipc_namespace *ns;
	DEFINE_WAKE_Q(wake_q);

	ns = current->nsproxy->ipc_ns;

	if (msgsz > ns->msg_ctlmax || (long) msgsz < 0 || msqid < 0)
		return -EINVAL;
	if (mtype < 1)
		return -EINVAL;

	msg = load_msg(mtext, msgsz); //构造msg_msg
	if (IS_ERR(msg))
		return PTR_ERR(msg);

	msg->m_type = mtype;
	msg->m_ts = msgsz;

	rcu_read_lock();
	msq = msq_obtain_object_check(ns, msqid);
	if (IS_ERR(msq)) {
		err = PTR_ERR(msq);
		goto out_unlock1;
	}

	ipc_lock_object(&msq->q_perm); //通过信号量同步

	for (;;) { //如果是阻塞式的一直循环尝试
		struct msg_sender s;

		err = -EACCES;
		if (ipcperms(ns, &msq->q_perm, S_IWUGO))
			goto out_unlock0;

		/* raced with RMID? */
		if (!ipc_valid_object(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}

		err = security_msg_queue_msgsnd(&msq->q_perm, msg, msgflg);
		if (err)
			goto out_unlock0;

		if (msg_fits_inqueue(msq, msgsz))
			break;

		/* queue full, wait: */
		if (msgflg & IPC_NOWAIT) {
			err = -EAGAIN;
			goto out_unlock0;
		}

		/* enqueue the sender and prepare to block */
		ss_add(msq, &s, msgsz);

		if (!ipc_rcu_getref(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}

		ipc_unlock_object(&msq->q_perm);
		rcu_read_unlock();
		schedule();

		rcu_read_lock();
		ipc_lock_object(&msq->q_perm);

		ipc_rcu_putref(&msq->q_perm, msg_rcu_free);
		/* raced with RMID? */
		if (!ipc_valid_object(&msq->q_perm)) {
			err = -EIDRM;
			goto out_unlock0;
		}
		ss_del(&s);

		if (signal_pending(current)) {
			err = -ERESTARTNOHAND;
			goto out_unlock0;
		}

	}

	ipc_update_pid(&msq->q_lspid, task_tgid(current));
	msq->q_stime = ktime_get_real_seconds();

	if (!pipelined_send(msq, msg, &wake_q)) {
		/* no one is waiting for this message, enqueue it */
		list_add_tail(&msg->m_list, &msq->q_messages);
		msq->q_cbytes += msgsz;
		msq->q_qnum++;
		percpu_counter_add_local(&ns->percpu_msg_bytes, msgsz);
		percpu_counter_add_local(&ns->percpu_msg_hdrs, 1);
	}

	err = 0;
	msg = NULL;
out_unlock0:
	ipc_unlock_object(&msq->q_perm);
	wake_up_q(&wake_q);
out_unlock1:
	rcu_read_unlock();
	if (msg != NULL)
		free_msg(msg);
	return err;
}
```
- 接收消息msgrcv
```c
SYSCALL_DEFINE5(msgrcv, int, msqid, struct msgbuf __user *, msgp, size_t, msgsz,
		long, msgtyp, int, msgflg)
{
	return ksys_msgrcv(msqid, msgp, msgsz, msgtyp, msgflg);
}
long ksys_msgrcv(int msqid, struct msgbuf __user *msgp, size_t msgsz,
		 long msgtyp, int msgflg)
{
	return do_msgrcv(msqid, msgp, msgsz, msgtyp, msgflg, do_msg_fill); //在这里读，和do_msgsnd的相反从操作
}
```

### 共享内存shared memory
共享内存的代码在`ipc\shm.c`
```c
struct shmid_kernel /* private to the kernel */
{
	struct kern_ipc_perm	shm_perm; //ipc权限相关
	struct file		*shm_file; //映射的内核文件，/proc/pid/maps.
	unsigned long		shm_nattch; //attach的进程数
	unsigned long		shm_segsz; //内存大小
	time64_t		shm_atim;
	time64_t		shm_dtim;
	time64_t		shm_ctim;
	struct pid		*shm_cprid; //创建进程
	struct pid		*shm_lprid; //最后操作进程
	struct ucounts		*mlock_ucounts;

	/*
	 * The task created the shm object, for
	 * task_lock(shp->shm_creator)
	 */
	struct task_struct	*shm_creator;

	/*
	 * List by creator. task_lock(->shm_creator) required for read/write.
	 * If list_empty(), then the creator is dead already.
	 */
	struct list_head	shm_clist;
	struct ipc_namespace	*ns;
} __randomize_layout;
```

- 创建共享内存newseg
```c
static int newseg(struct ipc_namespace *ns, struct ipc_params *params)
{
	key_t key = params->key;
	int shmflg = params->flg;
	size_t size = params->u.size;
	int error;
	struct shmid_kernel *shp;
	size_t numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT;
	struct file *file;
	char name[13];
	vm_flags_t acctflag = 0;

	if (size < SHMMIN || size > ns->shm_ctlmax)
		return -EINVAL;

	if (numpages << PAGE_SHIFT < size)
		return -ENOSPC;

	if (ns->shm_tot + numpages < ns->shm_tot ||
			ns->shm_tot + numpages > ns->shm_ctlall)
		return -ENOSPC;

	shp = kmalloc(sizeof(*shp), GFP_KERNEL_ACCOUNT);
	if (unlikely(!shp))
		return -ENOMEM;

	shp->shm_perm.key = key;
	shp->shm_perm.mode = (shmflg & S_IRWXUGO);
	shp->mlock_ucounts = NULL;

	shp->shm_perm.security = NULL;
	error = security_shm_alloc(&shp->shm_perm);
	if (error) {
		kfree(shp);
		return error;
	}

	sprintf(name, "SYSV%08x", key);
	if (shmflg & SHM_HUGETLB) {
		struct hstate *hs;
		size_t hugesize;

		hs = hstate_sizelog((shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK);
		if (!hs) {
			error = -EINVAL;
			goto no_file;
		}
		hugesize = ALIGN(size, huge_page_size(hs));

		/* hugetlb_file_setup applies strict accounting */
		if (shmflg & SHM_NORESERVE)
			acctflag = VM_NORESERVE;
		file = hugetlb_file_setup(name, hugesize, acctflag,
				HUGETLB_SHMFS_INODE, (shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK);
	} else {
		/*
		 * Do not allow no accounting for OVERCOMMIT_NEVER, even
		 * if it's asked for.
		 */
		if  ((shmflg & SHM_NORESERVE) &&
				sysctl_overcommit_memory != OVERCOMMIT_NEVER)
			acctflag = VM_NORESERVE;
		file = shmem_kernel_file_setup(name, size, acctflag);
	}
	error = PTR_ERR(file);
	if (IS_ERR(file))
		goto no_file;

	shp->shm_cprid = get_pid(task_tgid(current));
	shp->shm_lprid = NULL;
	shp->shm_atim = shp->shm_dtim = 0;
	shp->shm_ctim = ktime_get_real_seconds();
	shp->shm_segsz = size;
	shp->shm_nattch = 0;
	shp->shm_file = file;
	shp->shm_creator = current;

	/* ipc_addid() locks shp upon success. */
	error = ipc_addid(&shm_ids(ns), &shp->shm_perm, ns->shm_ctlmni);
	if (error < 0)
		goto no_id;

	shp->ns = ns;

	task_lock(current);
	list_add(&shp->shm_clist, ¤t->sysvshm.shm_clist);
	task_unlock(current);

	/*
	 * shmid gets reported as "inode#" in /proc/pid/maps.
	 * proc-ps tools use this. Changing this will break them.
	 */
	file_inode(file)->i_ino = shp->shm_perm.id;

	ns->shm_tot += numpages;
	error = shp->shm_perm.id;

	ipc_unlock_object(&shp->shm_perm);
	rcu_read_unlock();
	return error;
no_id:
	ipc_update_pid(&shp->shm_cprid, NULL);
	ipc_update_pid(&shp->shm_lprid, NULL);
	fput(file);
	ipc_rcu_putref(&shp->shm_perm, shm_rcu_free);
	return error;
no_file:
	call_rcu(&shp->shm_perm.rcu, shm_rcu_free);
	return error;
}
```

- 内存地址映射shmat
```c
SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
	unsigned long ret;
	long err;

	err = do_shmat(shmid, shmaddr, shmflg, &ret, SHMLBA);
	if (err)
		return err;
	force_successful_syscall_return();
	return (long)ret;
}
long do_shmat(int shmid, char __user *shmaddr, int shmflg,
	      ulong *raddr, unsigned long shmlba)
{
	struct shmid_kernel *shp;
	unsigned long addr = (unsigned long)shmaddr;
	unsigned long size;
	struct file *file, *base;
	int    err;
	unsigned long flags = MAP_SHARED;
	unsigned long prot;
	int acc_mode;
	struct ipc_namespace *ns;
	struct shm_file_data *sfd;
	int f_flags;
	unsigned long populate = 0;

	err = -EINVAL;
	if (shmid < 0)
		goto out;

	if (addr) {
		if (addr & (shmlba - 1)) {
			if (shmflg & SHM_RND) {
				addr &= ~(shmlba - 1);  /* round down */

				/*
				 * Ensure that the round-down is non-nil
				 * when remapping. This can happen for
				 * cases when addr < shmlba.
				 */
				if (!addr && (shmflg & SHM_REMAP))
					goto out;
			} else
#ifndef __ARCH_FORCE_SHMLBA
				if (addr & ~PAGE_MASK)
#endif
					goto out;
		}

		flags |= MAP_FIXED;
	} else if ((shmflg & SHM_REMAP))
		goto out;

	if (shmflg & SHM_RDONLY) {
		prot = PROT_READ;
		acc_mode = S_IRUGO;
		f_flags = O_RDONLY;
	} else {
		prot = PROT_READ | PROT_WRITE;
		acc_mode = S_IRUGO | S_IWUGO;
		f_flags = O_RDWR;
	}
	if (shmflg & SHM_EXEC) {
		prot |= PROT_EXEC;
		acc_mode |= S_IXUGO;
	}

	/*
	 * We cannot rely on the fs check since SYSV IPC does have an
	 * additional creator id...
	 */
	ns = current->nsproxy->ipc_ns;
	rcu_read_lock();
	shp = shm_obtain_object_check(ns, shmid);
	if (IS_ERR(shp)) {
		err = PTR_ERR(shp);
		goto out_unlock;
	}

	err = -EACCES;
	if (ipcperms(ns, &shp->shm_perm, acc_mode))
		goto out_unlock;

	err = security_shm_shmat(&shp->shm_perm, shmaddr, shmflg);
	if (err)
		goto out_unlock;

	ipc_lock_object(&shp->shm_perm);

	/* check if shm_destroy() is tearing down shp */
	if (!ipc_valid_object(&shp->shm_perm)) {
		ipc_unlock_object(&shp->shm_perm);
		err = -EIDRM;
		goto out_unlock;
	}

	/*
	 * We need to take a reference to the real shm file to prevent the
	 * pointer from becoming stale in cases where the lifetime of the outer
	 * file extends beyond that of the shm segment.  It's not usually
	 * possible, but it can happen during remap_file_pages() emulation as
	 * that unmaps the memory, then does ->mmap() via file reference only.
	 * We'll deny the ->mmap() if the shm segment was since removed, but to
	 * detect shm ID reuse we need to compare the file pointers.
	 */
	base = get_file(shp->shm_file);
	shp->shm_nattch++;
	size = i_size_read(file_inode(base));
	ipc_unlock_object(&shp->shm_perm);
	rcu_read_unlock();

	err = -ENOMEM;
	sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);
	if (!sfd) {
		fput(base);
		goto out_nattch;
	}

	file = alloc_file_clone(base, f_flags,
			  is_file_hugepages(base) ?
				&shm_file_operations_huge :
				&shm_file_operations);
	err = PTR_ERR(file);
	if (IS_ERR(file)) {
		kfree(sfd);
		fput(base);
		goto out_nattch;
	}

	sfd->id = shp->shm_perm.id;
	sfd->ns = get_ipc_ns(ns);
	sfd->file = base;
	sfd->vm_ops = NULL;
	file->private_data = sfd;

	err = security_mmap_file(file, prot, flags);
	if (err)
		goto out_fput;

	if (mmap_write_lock_killable(current->mm)) {
		err = -EINTR;
		goto out_fput;
	}

	if (addr && !(shmflg & SHM_REMAP)) {
		err = -EINVAL;
		if (addr + size < addr)
			goto invalid;

		if (find_vma_intersection(current->mm, addr, addr + size))
			goto invalid;
	}

	addr = do_mmap(file, addr, size, prot, flags, 0, 0, &populate, NULL);
	*raddr = addr;
	err = 0;
	if (IS_ERR_VALUE(addr))
		err = (long)addr;
invalid:
	mmap_write_unlock(current->mm);
	if (populate)
		mm_populate(addr, populate);
out_fput:
	fput(file);
out_nattch:
	down_write(&shm_ids(ns).rwsem);
	shp = shm_lock(ns, shmid);
	shp->shm_nattch--;

	if (shm_may_destroy(shp))
		shm_destroy(ns, shp);
	else
		shm_unlock(shp);
	up_write(&shm_ids(ns).rwsem);
	return err;
out_unlock:
	rcu_read_unlock();
out:
	return err;
}
```
