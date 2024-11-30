### 协议规范

- IPv4
  IP报文头固定部分长度是20字节，可选内容Options里是TLV列表，现在一般不太用。详情参见[https://www.rfc-editor.org/rfc/rfc791](https://www.rfc-editor.org/rfc/rfc791 "https://www.rfc-editor.org/rfc/rfc791")
![ipv4-header.png](https://ping666.com/wp-content/uploads/2024/09/ipv4-header.png "ipv4-header.png")
对应内核结构iphdr
```c
struct iphdr {
#if defined(__LITTLE_ENDIAN_BITFIELD)
	__u8	ihl:4,
		version:4;
#elif defined (__BIG_ENDIAN_BITFIELD)
	__u8	version:4,
  		ihl:4;
#else
#error	"Please fix <asm/byteorder.h>"
#endif
	__u8	tos;
	__be16	tot_len;
	__be16	id;
	__be16	frag_off;
	__u8	ttl;
	__u8	protocol;
	__sum16	check;
	__struct_group(/* no tag */, addrs, /* no attrs */,
		__be32	saddr;
		__be32	daddr;
	);
	/*The options start here. */
};
```
- UDP
  UDP报文头长度是固定的，8字节。详情参见[https://www.rfc-editor.org/rfc/rfc768](https://www.rfc-editor.org/rfc/rfc768 "https://www.rfc-editor.org/rfc/rfc768")
![udp-header.png](https://ping666.com/wp-content/uploads/2024/09/udp-header.png "udp-header.png")
对应内核结构udphdr
```c
struct udphdr {
	__be16	source;
	__be16	dest;
	__be16	len;
	__sum16	check;
};
```
- TCP
  TCP报文头固定部分长度是20字节。详情参见
  ![tcp-header.png](https://ping666.com/wp-content/uploads/2024/09/tcp-header.png "tcp-header.png")
对应内核结构tcphdr
```c
struct tcphdr {
	__be16	source;
	__be16	dest;
	__be32	seq;
	__be32	ack_seq;
#if defined(__LITTLE_ENDIAN_BITFIELD)
	__u16	res1:4,
		doff:4,
		fin:1,
		syn:1,
		rst:1,
		psh:1,
		ack:1,
		urg:1,
		ece:1,
		cwr:1;
#elif defined(__BIG_ENDIAN_BITFIELD)
	__u16	doff:4,
		res1:4,
		cwr:1,
		ece:1,
		urg:1,
		ack:1,
		psh:1,
		rst:1,
		syn:1,
		fin:1;
#else
#error	"Adjust your <asm/byteorder.h> defines"
#endif	
	__be16	window;
	__sum16	check;
	__be16	urg_ptr;
};
```

### 一切皆文件
socket也是作为文件来操作的。入口代码在`net\socket.c`
```c
static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.uring_cmd =    io_uring_cmd_sock,
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.splice_write = splice_to_socket,
	.splice_read =	sock_splice_read,
	.splice_eof =	sock_splice_eof,
	.show_fdinfo =	sock_show_fdinfo,
};
```

协议栈一些核心数据结构之间的关系
![linux-net.png](https://ping666.com/wp-content/uploads/2024/09/linux-net.png "linux-net.png")

### UDP初始化
udp的初始化函数在`net\ipv4\udp.c`
```c
void __init udp_init(void)
{
	unsigned long limit;
	unsigned int i;

	udp_table_init(&udp_table, "UDP");
	limit = nr_free_buffer_pages() / 8;
	limit = max(limit, 128UL);
	sysctl_udp_mem[0] = limit / 4 * 3;
	sysctl_udp_mem[1] = limit;
	sysctl_udp_mem[2] = sysctl_udp_mem[0] * 2;

	/* 16 spinlocks per cpu */
	udp_busylocks_log = ilog2(nr_cpu_ids) + 4;
	udp_busylocks = kmalloc(sizeof(spinlock_t) << udp_busylocks_log,
				GFP_KERNEL);
	if (!udp_busylocks)
		panic("UDP: failed to alloc udp_busylocks\n");
	for (i = 0; i < (1U << udp_busylocks_log); i++)
		spin_lock_init(udp_busylocks + i);

	if (register_pernet_subsys(&udp_sysctl_ops))
		panic("UDP: failed to init sysctl parameters.\n");

#if defined(CONFIG_BPF_SYSCALL) && defined(CONFIG_PROC_FS)
	bpf_iter_register();
#endif
}
```

udp_init是作为AF_INIT协议族（inet_init）的一部分初始化的，还包括其他协议如arp、ip、tcp、icmp等。
inet_init是内核定义的一个initcall。
`fs_initcall(inet_init);`
所有的initcall定义如下
```c
#define pure_initcall(fn)		__define_initcall(fn, 0)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5) //在5这个顺序
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

所有的initcall会在内核初始化kernel_init函数里调用（在CPU、内存、进程、外设、中断子系统之后）。
```c
static void __init do_initcalls(void)
{
	int level;
	size_t len = saved_command_line_len + 1;
	char *command_line;

	command_line = kzalloc(len, GFP_KERNEL);
	if (!command_line)
		panic("%s: Failed to allocate %zu bytes\n", __func__, len);

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
		/* Parser modifies command_line, restore it each time */
		strcpy(command_line, saved_command_line);
		do_initcall_level(level, command_line);
	}

	kfree(command_line);
}
```

### udp_sendmsg
UDP发送为了提高性能，用了一些技巧

- RSP
  Receive Packet Steering，实现CPU负载均衡
- XDP
  eXpress Data Path，和eBPF（Extended Berkeley Packet Filter）配合使用，避免内核态和用户态的内存拷贝
- Corking
  小包合并

详细代码如下`net\ipv4\udp.c`
```c
int udp_sendmsg(struct sock *sk, struct msghdr *msg, size_t len)
{
	struct inet_sock *inet = inet_sk(sk); //拿到sk所在的inet_sock
	struct udp_sock *up = udp_sk(sk); //拿到sk所在的udp_sock
    DECLARE_SOCKADDR(struct sockaddr_in *, usin, msg->msg_name);
    ......
	if (len > 0xFFFF) // 64k
		return -EMSGSIZE;

	/*
	 *	Check the flags.
	 */

	if (msg->msg_flags & MSG_OOB) /* Mirror BSD error message compatibility */
		return -EOPNOTSUPP;

	getfrag = is_udplite ? udplite_getfrag : ip_generic_getfrag; // udp lite

	fl4 = &inet->cork.fl.u.ip4;
	if (READ_ONCE(up->pending)) {
		/*
		 * There are pending frames.
		 * The socket lock must be held while it's corked.
		 */
		lock_sock(sk);
		if (likely(up->pending)) {
			if (unlikely(up->pending != AF_INET)) {
				release_sock(sk);
				return -EINVAL;
			}
			goto do_append_data; //socket上已经有发送数据直接追加在后面
		}
		release_sock(sk);
	}
	ulen += sizeof(struct udphdr);

	/*
	 *	Get and verify the address.
	 */
	if (usin) {
		if (msg->msg_namelen < sizeof(*usin))
			return -EINVAL;
		if (usin->sin_family != AF_INET) {
			if (usin->sin_family != AF_UNSPEC)
				return -EAFNOSUPPORT;
		}

		daddr = usin->sin_addr.s_addr;
		dport = usin->sin_port;
		if (dport == 0)
			return -EINVAL;
	} else {
		if (sk->sk_state != TCP_ESTABLISHED)
			return -EDESTADDRREQ;
		daddr = inet->inet_daddr;
		dport = inet->inet_dport;
		/* Open fast path for connected socket.
		   Route will not be used, if at least one option is set.
		 */
		connected = 1;
	}

	ipcm_init_sk(&ipc, inet);
	ipc.gso_size = READ_ONCE(up->gso_size);

	if (msg->msg_controllen) {
		err = udp_cmsg_send(sk, msg, &ipc.gso_size);
		if (err > 0) {
			err = ip_cmsg_send(sk, msg, &ipc,
					   sk->sk_family == AF_INET6);
			connected = 0;
		}
		if (unlikely(err < 0)) {
			kfree(ipc.opt);
			return err;
		}
		if (ipc.opt)
			free = 1;
	}
	if (!ipc.opt) {
		struct ip_options_rcu *inet_opt;

		rcu_read_lock();
		inet_opt = rcu_dereference(inet->inet_opt);
		if (inet_opt) {
			memcpy(&opt_copy, inet_opt,
			       sizeof(*inet_opt) + inet_opt->opt.optlen);
			ipc.opt = &opt_copy.opt;
		}
		rcu_read_unlock();
	}

	if (cgroup_bpf_enabled(CGROUP_UDP4_SENDMSG) && !connected) { //支持bpf
		err = BPF_CGROUP_RUN_PROG_UDP4_SENDMSG_LOCK(sk,
					    (struct sockaddr *)usin,
					    &msg->msg_namelen,
					    &ipc.addr);
		if (err)
			goto out_free;
		if (usin) {
			if (usin->sin_port == 0) {
				/* BPF program set invalid port. Reject it. */
				err = -EINVAL;
				goto out_free;
			}
			daddr = usin->sin_addr.s_addr;
			dport = usin->sin_port;
		}
	}

	saddr = ipc.addr;
	ipc.addr = faddr = daddr;

	if (ipc.opt && ipc.opt->opt.srr) {
		if (!daddr) {
			err = -EINVAL;
			goto out_free;
		}
		faddr = ipc.opt->opt.faddr;
		connected = 0;
	}
	tos = get_rttos(&ipc, inet);
	scope = ip_sendmsg_scope(inet, &ipc, msg);
	if (scope == RT_SCOPE_LINK)
		connected = 0;

	uc_index = READ_ONCE(inet->uc_index);
	if (ipv4_is_multicast(daddr)) { //广播地址
		if (!ipc.oif || netif_index_is_l3_master(sock_net(sk), ipc.oif))
			ipc.oif = READ_ONCE(inet->mc_index);
		if (!saddr)
			saddr = READ_ONCE(inet->mc_addr);
		connected = 0;
	} else if (!ipc.oif) {
		ipc.oif = uc_index;
	} else if (ipv4_is_lbcast(daddr) && uc_index) {
		/* oif is set, packet is to local broadcast and
		 * uc_index is set. oif is most likely set
		 * by sk_bound_dev_if. If uc_index != oif check if the
		 * oif is an L3 master and uc_index is an L3 slave.
		 * If so, we want to allow the send using the uc_index.
		 */
		if (ipc.oif != uc_index &&
		    ipc.oif == l3mdev_master_ifindex_by_index(sock_net(sk),
							      uc_index)) {
			ipc.oif = uc_index;
		}
	}

	if (connected)
		rt = dst_rtable(sk_dst_check(sk, 0));

	if (!rt) { //没有路由则新建一个
		struct net *net = sock_net(sk);
		__u8 flow_flags = inet_sk_flowi_flags(sk);

		fl4 = &fl4_stack;

		flowi4_init_output(fl4, ipc.oif, ipc.sockc.mark, tos, scope,
				   sk->sk_protocol, flow_flags, faddr, saddr,
				   dport, inet->inet_sport, sk->sk_uid);

		security_sk_classify_flow(sk, flowi4_to_flowi_common(fl4));
		rt = ip_route_output_flow(net, fl4, sk);
		if (IS_ERR(rt)) {
			err = PTR_ERR(rt);
			rt = NULL;
			if (err == -ENETUNREACH)
				IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
			goto out;
		}

		err = -EACCES;
		if ((rt->rt_flags & RTCF_BROADCAST) &&
		    !sock_flag(sk, SOCK_BROADCAST))
			goto out;
		if (connected)
			sk_dst_set(sk, dst_clone(&rt->dst));
	}

	if (msg->msg_flags&MSG_CONFIRM)
		goto do_confirm;
back_from_confirm:

	saddr = fl4->saddr;
	if (!ipc.addr)
		daddr = ipc.addr = fl4->daddr;

	/* Lockless fast path for the non-corking case. */
	if (!corkreq) { //合并多个小包
		struct inet_cork cork;

		skb = ip_make_skb(sk, fl4, getfrag, msg, ulen,
				  sizeof(struct udphdr), &ipc, &rt,
				  &cork, msg->msg_flags);
		err = PTR_ERR(skb);
		if (!IS_ERR_OR_NULL(skb))
			err = udp_send_skb(skb, fl4, &cork); //发送包
		goto out;
	}

	lock_sock(sk);
	if (unlikely(up->pending)) {
		/* The socket is already corked while preparing it. */
		/* ... which is an evident application bug. --ANK */
		release_sock(sk);

		net_dbg_ratelimited("socket already corked\n");
		err = -EINVAL;
		goto out;
	}
	/*
	 *	Now cork the socket to pend data.
	 */
	fl4 = &inet->cork.fl.u.ip4;
	fl4->daddr = daddr;
	fl4->saddr = saddr;
	fl4->fl4_dport = dport;
	fl4->fl4_sport = inet->inet_sport;
	WRITE_ONCE(up->pending, AF_INET);

do_append_data:
	up->len += ulen;
	err = ip_append_data(sk, fl4, getfrag, msg, ulen,
			     sizeof(struct udphdr), &ipc, &rt,
			     corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
	if (err)
		udp_flush_pending_frames(sk);
	else if (!corkreq)
		err = udp_push_pending_frames(sk);
	else if (unlikely(skb_queue_empty(&sk->sk_write_queue)))
		WRITE_ONCE(up->pending, 0);
	release_sock(sk);

out:
	ip_rt_put(rt);
out_free:
	if (free)
		kfree(ipc.opt);
	if (!err)
		return len;
	/*
	 * ENOBUFS = no kernel mem, SOCK_NOSPACE = no sndbuf space.  Reporting
	 * ENOBUFS might not be good (it's not tunable per se), but otherwise
	 * we don't have a good statistic (IpOutDiscards but it can be too many
	 * things).  We could add another new stat but at least for now that
	 * seems like overkill.
	 */
	if (err == -ENOBUFS || test_bit(SOCK_NOSPACE, &sk->sk_socket->flags)) {
		UDP_INC_STATS(sock_net(sk),
			      UDP_MIB_SNDBUFERRORS, is_udplite);
	}
	return err;

do_confirm:
	if (msg->msg_flags & MSG_PROBE)
		dst_confirm_neigh(&rt->dst, &fl4->daddr);
	if (!(msg->msg_flags&MSG_PROBE) || len)
		goto back_from_confirm;
	err = 0;
	goto out;
}
EXPORT_SYMBOL(udp_sendmsg);
```

### udp_recvmsg
udp_recvmsg的代码结构类似，但是流程更简单一点。从udp_sock.reader_queue队列上取sk_buff返回给用户接口。

![linux-udp.png](https://ping666.com/wp-content/uploads/2024/09/linux-udp.png "linux-udp.png")

### TCP和极致的快
TCP协议和UDP工作在同一层级，只是多了window滑动窗口，拥塞控制，慢启动，快速重传和快恢复等一系列流量控制的手段。但是这一套机制都是基于丢包率高，网络环境比较差的前提下给出的解决方案。现在的网络无论是带宽，延迟，还是丢包率都已有质的飞跃。所以UDP协议正越来越多的被使用。

为了追求极致的网络性能，Intel还提供了DPDK（Intel Data Plane Development Kit）方案，该方案在量化交易等场景下被大量使用。
![dpdk.png](https://ping666.com/wp-content/uploads/2024/10/dpdk.png "dpdk.png")

DPDK是基于Linux的DMA（Direct Memory Access，直接内存访问）机制，绕过内核在用户态直接处理网络数据。

- Kenel Bypass，避免大量的系统调用切换
- PMD(轮询模式驱动)，使用轮询代替系统中断
- Zero Copy，数据直接从硬件到用户态避免多次拷贝
- 以及更多关于内存、协议栈的优化

Linux提供了一种类似的Kenel Bypass机制，eBPF。可以动态插入回调代码到内核中，也是非常强悍。

### 侵入式链表list_head

- 非侵入式链表
一般的链表都是非侵入式的，大部分传统编程语言中的链表都是下面这样的。链表的数据内容E和链表本身的节点结构Node是完全独立，解耦的。这在设计上是一个好的模式，但是在性能上却不是极致。
```java
class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```

- 侵入式链表
Linix内核中的链表普遍是下面这样的，链表的数据内容E上有节点指针list_head。链表遍历时通过first/last结构的container_of操作，拿到链表节点绑定的数据内容。这种侵入式的链表结构在内存结构和CPU缓存中都到来了更多的好处。使性能得到大幅提升。当然这一切的关键是container_of宏。
```c
struct E
{
    struct list_head* first;
    struct list_head* last;
    ......
}
```

- container_of宏
  通过属性字段拿到包含他的结构。内核接口参数统一为通用的sock，但是在接口内部实现里可以通过sokc拿到实际的类型如inet_sock、udp_sock，达到类似Java里instanceof的效果。使用的是GCC编译器特性__builtin_offsetof。
```c
#define inet_sk(ptr) container_of_const(ptr, struct inet_sock, sk)
#define udp_sk(ptr) container_of_const(ptr, struct udp_sock, inet.sk)
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	static_assert(__same_type(*(ptr), ((type *)0)->member) ||	\
		      __same_type(*(ptr), void),			\
		      "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })
```
