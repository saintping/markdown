### 基本权限控制
用户权限这块，传统的Unix-Like操作系统都使用自主访问控制（DAC，Discretionary Access Control），简单点说就是拿到root权限就可以干任何事。
```bash
#ls -alh
-rw-r--r--   1 root root 314K Jun  4 22:48 symvers-3.10.0-1160.119.1.el7.x86_64.gz
-rw-------   1 root root 3.5M Jun  4 22:48 System.map-3.10.0-1160.119.1.el7.x86_64
-rwxr-xr-x   1 root root 6.8M Jun  4 22:48 vmlinuz-3.10.0-1160.119.1.el7.x86_64
```

以vmlinuz-3.10.0-1160.119.1.el7.x86_64为例。这里的权限位`-rwxr-xr-x`，第一位代表是否目录d，后面三位是文件属主的权限rwx（可读、可写、可执行），中间三位是本组其他账号的权限r-x（可读、可执行），最后三位是其他用户的权限r-x（可读、可执行）。

- 可读
  r，数值表示4
- 可写
  w，数值表示2
- 可执行
  x，数值表示1
- S位
  s，其他账号有和文件属组一样的权限，典型的就是/bin/passwd和/usr/bin/sudo

通过chown和chgrp可以修改文件的用户和组。

当权限不够时，经常暴力的chmod 777，就是权限全部加上，这是很危险的。因为root权限太高，所以一般软件（如Ningx、Apache、Mysql）安装时都会默认创建一个用户，避免直接使用root账号。

Linux内核的基本权限控制在函数inode_permission里（在系统调用之后，调用实际文件系统接口之前）。代码在`fs\namei.c`
```c
int inode_permission(struct mnt_idmap *idmap,
		     struct inode *inode, int mask)
{
	int retval;

	retval = sb_permission(inode->i_sb, inode, mask);
	if (retval)
		return retval;

	if (unlikely(mask & MAY_WRITE)) {
		/*
		 * Nobody gets write access to an immutable file.
		 */
		if (IS_IMMUTABLE(inode))
			return -EPERM;

		/*
		 * Updating mtime will likely cause i_uid and i_gid to be
		 * written back improperly if their true value is unknown
		 * to the vfs.
		 */
		if (HAS_UNMAPPED_ID(idmap, inode))
			return -EACCES;
	}

	retval = do_inode_permission(idmap, inode, mask);
	if (retval)
		return retval;

	retval = devcgroup_inode_permission(inode, mask);
	if (retval)
		return retval;

	return security_inode_permission(inode, mask);
}
```

### Selinux
上面的DAC权限控制，最怕的就是被提权。所以SELinux（Security-Enhanced Linux）搞了一套强制访问控制（MAC，Mandatory Access Control），采用最小授权原则，通过额外的配置文件来显式规定用户能干什么。

Selinux默认是关闭的：
```bash
cat /etc/sysconfig/selinux
SELINUX=disabled
```

开启了之后就可以对系统进行规则配置setsebool和上下文配置semanage。
Selinux和文件的基本权限控制是独立的，可以配合使用。但是因为配置过于复杂，大部分情况下都是关闭。Selinux已经被容器技术降维打击了，不继续看代码。

Docker容器技术三大利器：Namespace + CGroup + UnionFS，其中Namespace和CGroup都是Linux内核的特性。

Linux内核的用户资源隔离有两个层面：

- 进程可以看见什么，Namespace
- 进程可以使用多少，CGroup

### Namespace
和高级语言中的命名空间类似，Namespace也是为了可见性的问题。命名空间内的资源只对这个命名空间内的进程可见，对其他进程是透明的。

Namespace是很后面才加入的特性，到2.6版本才比较完善。PCB结构上加了字段
`struct nsproxy			*nsproxy;`

支持Namespace的子系统有用户组、进程、进程间通信、网络、文件系统等。
```c
struct nsproxy {
	refcount_t count;
	struct uts_namespace *uts_ns; //主机和域名
	struct ipc_namespace *ipc_ns; //进程间通信资源：消息队列、信号、共享内存等
	struct mnt_namespace *mnt_ns; //文件挂载点
	struct pid_namespace *pid_ns_for_children; //进程ID
	struct net 	     *net_ns; //网络
	struct time_namespace *time_ns; //
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns; //cgroup根目录
};
```

每个子系统都加上了支持Namesapce的结构。比如下面的进程ID的命名空间。
```c
struct pid_namespace {
	struct idr idr; //ns下独立的pid管理
	struct rcu_head rcu;
	unsigned int pid_allocated; //ns下独立的pid管理
	struct task_struct *child_reaper;
	struct kmem_cache *pid_cachep; //ns下独立的pid管理
	unsigned int level;
	struct pid_namespace *parent;
#ifdef CONFIG_BSD_PROCESS_ACCT
	struct fs_pin *bacct;
#endif
	struct user_namespace *user_ns;
	struct ucounts *ucounts;
	int reboot;	/* group exit code if this pidns was rebooted */
	struct ns_common ns;
#if defined(CONFIG_SYSCTL) && defined(CONFIG_MEMFD_CREATE)
	int memfd_noexec_scope;
#endif
} __randomize_layout;
```

在/proc里可以确认每个进程的所属命名空间。
```bash
# ls -l /proc/1/ns/
lrwxrwxrwx 1 root root 0 Sep 14 12:37 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 uts -> uts:[4026531838]
# ls -l /proc/10/ns/
lrwxrwxrwx 1 root root 0 Sep 14 12:37 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Sep 14 12:37 uts -> uts:[4026531838]
```

### CGroup

- 整体结构
  命名空间解决了资源可见的问题，即进程能看见什么。但是没有解决能用多少的问题，比如进程运行起来后对CPU或者内存的抢占。CGroup（Control Group）就是用来解决这个问题的，对资源按量分配和调度。
![linux-cgroup.png](https://ping666.com/wp-content/uploads/2024/09/linux-cgroup.png "linux-cgroup.png")

- 配额管理
  几乎每个核心的子系统都支持CGroup，有他自己的配额管理。CPU的配额文件在`/sys/fs/cgroup/cpu`，这下面的每个目录都对应一个cgroup结构，由cgroup_create函数创建。代码在`kernel\cgroup\cgroup.c`
```c
static struct cgroup *cgroup_create(struct cgroup *parent, const char *name,
				    umode_t mode) // name是本次cgroup对应的目录名
{
	struct cgroup_root *root = parent->root;
	struct cgroup *cgrp, *tcgrp;
	struct kernfs_node *kn;
	int level = parent->level + 1;
	int ret;

	/* allocate the cgroup and its ID, 0 is reserved for the root */
	cgrp = kzalloc(struct_size(cgrp, ancestors, (level + 1)), GFP_KERNEL); //创建新的cgroup
	if (!cgrp)
		return ERR_PTR(-ENOMEM);

	ret = percpu_ref_init(&cgrp->self.refcnt, css_release, 0, GFP_KERNEL);
	if (ret)
		goto out_free_cgrp;

	ret = cgroup_rstat_init(cgrp); //cpu状态
	if (ret)
		goto out_cancel_ref;

	/* create the directory */
	kn = kernfs_create_dir_ns(parent->kn, name, mode,
				  current_fsuid(), current_fsgid(),
				  cgrp, NULL); //创建cgroup对应的目录
	if (IS_ERR(kn)) {
		ret = PTR_ERR(kn);
		goto out_stat_exit;
	}
	cgrp->kn = kn;

	init_cgroup_housekeeping(cgrp);

	cgrp->self.parent = &parent->self;
	cgrp->root = root;
	cgrp->level = level;

	ret = psi_cgroup_alloc(cgrp);
	if (ret)
		goto out_kernfs_remove;

	ret = cgroup_bpf_inherit(cgrp);
	if (ret)
		goto out_psi_free;

	/*
	 * New cgroup inherits effective freeze counter, and
	 * if the parent has to be frozen, the child has too.
	 */
	cgrp->freezer.e_freeze = parent->freezer.e_freeze;
	if (cgrp->freezer.e_freeze) {
		/*
		 * Set the CGRP_FREEZE flag, so when a process will be
		 * attached to the child cgroup, it will become frozen.
		 * At this point the new cgroup is unpopulated, so we can
		 * consider it frozen immediately.
		 */
		set_bit(CGRP_FREEZE, &cgrp->flags);
		set_bit(CGRP_FROZEN, &cgrp->flags);
	}

	spin_lock_irq(&css_set_lock);
	for (tcgrp = cgrp; tcgrp; tcgrp = cgroup_parent(tcgrp)) {
		cgrp->ancestors[tcgrp->level] = tcgrp;

		if (tcgrp != cgrp) {
			tcgrp->nr_descendants++;

			/*
			 * If the new cgroup is frozen, all ancestor cgroups
			 * get a new frozen descendant, but their state can't
			 * change because of this.
			 */
			if (cgrp->freezer.e_freeze)
				tcgrp->freezer.nr_frozen_descendants++;
		}
	}
	spin_unlock_irq(&css_set_lock);

	if (notify_on_release(parent))
		set_bit(CGRP_NOTIFY_ON_RELEASE, &cgrp->flags);

	if (test_bit(CGRP_CPUSET_CLONE_CHILDREN, &parent->flags))
		set_bit(CGRP_CPUSET_CLONE_CHILDREN, &cgrp->flags);

	cgrp->self.serial_nr = css_serial_nr_next++;

	/* allocation complete, commit to creation */
	list_add_tail_rcu(&cgrp->self.sibling, &cgroup_parent(cgrp)->self.children);
	atomic_inc(&root->nr_cgrps);
	cgroup_get_live(parent);

	/*
	 * On the default hierarchy, a child doesn't automatically inherit
	 * subtree_control from the parent.  Each is configured manually.
	 */
	if (!cgroup_on_dfl(cgrp))
		cgrp->subtree_control = cgroup_control(cgrp);

	cgroup_propagate_control(cgrp);

	return cgrp;
out_psi_free:
	psi_cgroup_free(cgrp);
out_kernfs_remove:
	kernfs_remove(cgrp->kn);
out_stat_exit:
	cgroup_rstat_exit(cgrp);
out_cancel_ref:
	percpu_ref_exit(&cgrp->self.refcnt);
out_free_cgrp:
	kfree(cgrp);
	return ERR_PTR(ret);
}
```

- 配额使用
  这些配额值在调度的时候会被使用，比如CPU的公平调度策略`kernel\sched\fair.c`。
  其他目录下的配额比如`/sys/fs/cgroup/memory`，`/sys/fs/cgroup/blkio`也会在对应的子系统里被使用。
