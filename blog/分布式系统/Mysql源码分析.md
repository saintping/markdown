### Mysql架构
Mysql是一个开源的关系型数据库，项目主体为C++实现。自从卖身给Oracle之后，换了个马甲MariaDB继续开源。[https://github.com/MariaDB/server](https://github.com/MariaDB/server "https://github.com/MariaDB/server")
本文以MariaDB 10.0.0版本来回顾一下Mysql的源码。

![mysql-architecture.png](https://ping666.com/wp-content/uploads/2024/09/mysql-architecture.png "mysql-architecture.png")

Mysql是一个典型的单进程多线程服务。采用的是connection-per-thread模型，主线程负责进程环境初始化和网络监听，接收到新连接后交给单独线程去处理。

### mysqld启动过程
`sql\main.cc`中的main函数直接透传到`sql\mysqld.cc`里的mysqld_main。启动步骤如下：

1. load_defaults合并输入参数和my.cnf里的配置值
1. logger.init_base初始化日志
1. init_common_variables初始化全局参数
1. init_signals初始化系统信号处理函数
1. set_user设置用户组和用户
1. init_ssl加载SSL证书
1. network_init初始化网络端口和处理函数（mysql也支持单线程模型，在这里适配网络处理函数）
1. start_signal_handler处理控制信号函数、写pid文件、并且设置线程栈大小my_thread_stack_size= (sizeof(void*) <= 4)? 65536: ((256-16)*1024)
1. grant_init加载表和字段的授权信息
1. servers_init初始化锁、内存、缓存等
1. start_handle_manager启动管理线程
1. read_init_file读取init_file并且执行命令
1. 服务状态置为启动mysqld_server_started=1
1. handle_connections_sockets循环监听端口，接收连接并且创建新线程处理THD

THD是连接的核心数据结构，里面有这个连接相关的所有上下文数据。
以上只是mysql启动的主流程，还有些分支流程就不展开了。比如WRITE_DELAY（先回复客户端再写入）、EMBEDDED_LIBRARY（嵌入库形式）、WIN（适配windows）等。

### 命令执行
新线程的处理代码在`sql\sql_connect.cc`，入口是do_handle_one_connection。
```cpp
void do_handle_one_connection(THD *thd_arg){
  ......
  for (;;)
  {
    bool create_user= TRUE;

    mysql_socket_set_thread_owner(thd->net.vio->mysql_socket);
    if (thd_prepare_connection(thd))
    {
      create_user= FALSE;
      goto end_thread;
    }      

    while (thd_is_connection_alive(thd))
    {
      mysql_audit_release(thd);
      if (do_command(thd)) //这里读取网络包并且调用dispatch_command处理
        break;
    }
    end_connection(thd);
   
end_thread:
    close_connection(thd);

    if (thd->userstat_running)
      update_global_user_stats(thd, create_user, time(NULL));

    if (MYSQL_CALLBACK_ELSE(thd->scheduler, end_thread, (thd, 1), 0))
      return;                                 // Probably no-threads

    /*
      If end_thread() returns, this thread has been schedule to
      handle the next connection.
    */
    thd= current_thd;
    thd->thread_stack= (char*) &thd;
  }
}
```
dispatch_command的代码在`sql\sql_parse.cc`，里面是一个大switch(command)。mysql支持的所有命令（二进制协议）如下：
```cpp
enum enum_server_command
{
  COM_SLEEP, COM_QUIT, COM_INIT_DB, COM_QUERY, COM_FIELD_LIST,
  COM_CREATE_DB, COM_DROP_DB, COM_REFRESH, COM_SHUTDOWN, COM_STATISTICS,
  COM_PROCESS_INFO, COM_CONNECT, COM_PROCESS_KILL, COM_DEBUG, COM_PING,
  COM_TIME, COM_DELAYED_INSERT, COM_CHANGE_USER, COM_BINLOG_DUMP,
  COM_TABLE_DUMP, COM_CONNECT_OUT, COM_REGISTER_SLAVE,
  COM_STMT_PREPARE, COM_STMT_EXECUTE, COM_STMT_SEND_LONG_DATA, COM_STMT_CLOSE,
  COM_STMT_RESET, COM_SET_OPTION, COM_STMT_FETCH, COM_DAEMON,
  /* don't forget to update const char *command_name[] in sql_parse.cc */

  /* Must be last */
  COM_END
};
```
dispatch_command中会调用mysql_parse编译SQL。SQL编译这块采用yacc实现，Token文件在`sql\lex.h`，语法定义在`sql\sql_yacc.yy`。关于SQL编译这一块就不展开了，继续看SQL执行。

### SQL执行
以insert语句为例，会调用到`sql\sql_parse.cc`里的mysql_execute_command。这里也是一个大switch(lex->sql_command)。
```cpp
`int mysql_execute_command(THD *thd) {
  ......
  switch(lex->sql_command) {
  ......
  case SQLCOM_INSERT:
  {
    DBUG_ASSERT(first_table == all_tables && first_table != 0);
    if ((res= insert_precheck(thd, all_tables))) //检查权限
      break;

    MYSQL_INSERT_START(thd->query());
    res= mysql_insert(thd, all_tables, lex->field_list, lex->many_values,
		      lex->update_list, lex->value_list,
                      lex->duplicates, lex->ignore);
    MYSQL_INSERT_DONE(res, (ulong) thd->get_row_count_func());
    /*
      If we have inserted into a VIEW, and the base table has
      AUTO_INCREMENT column, but this column is not accessible through
      a view, then we should restore LAST_INSERT_ID to the value it
      had before the statement.
    */
    if (first_table->view && !first_table->contain_auto_increment)
      thd->first_successful_insert_id_in_cur_stmt=
        thd->first_successful_insert_id_in_prev_stmt;

    DBUG_EXECUTE_IF("after_mysql_insert",
                    {
                      const char act1[]=
                        "now "
                        "wait_for signal.continue";
                      const char act2[]=
                        "now "
                        "signal signal.continued";
                      DBUG_ASSERT(debug_sync_service);
                      DBUG_ASSERT(!debug_sync_set_action(thd,
                                                         STRING_WITH_LEN(act1)));
                      DBUG_ASSERT(!debug_sync_set_action(thd,
                                                         STRING_WITH_LEN(act2)));
                    };);
    DEBUG_SYNC(thd, "after_mysql_insert");
    break;
  }
  ......
}
```
`sql\sql_insert.cc`的mysql_insert实现如下：

1. upgrade_lock_type试着升级表锁
1. open_and_lock_tables加表结构锁
1. mysql_prepare_insert准备和检查字段值是否合法
1. fill_record_n_invoke_before_triggers触发器before_triggers
1. write_record循环调用存储引擎接口写每行数据（insert语句对应write_row/replace语句对应update_row）
1. binlog_log_row写binglog日志（binlog并不是存储引擎的功能，是server层的一部分）
1. my_ok返回消息给客户端

### 存储引擎
Mysql的存储引擎是插件化的，基类是handler，大约有100+个虚方法。
```cpp
class handler :public Sql_alloc
{
......
public:
    virtual handler *clone(const char *name, MEM_ROOT *mem_root);
    virtual int prepare_index_scan() { return 0; }
    virtual ha_rows records() { return stats.records; }
    virtual int rename_table(const char *from, const char *to);
    virtual int delete_table(const char *name);
    virtual int open(const char *name, int mode, uint test_if_locked)=0;
    virtual int index_init(uint idx, bool sorted) { return 0; }
    virtual int index_end() { return 0; }
    virtual ha_rows multi_range_read_info_const(uint keyno, RANGE_SEQ_IF *seq,
                                              void *seq_init_param, 
                                              uint n_ranges, uint *bufsz,
                                              uint *mrr_mode,
                                              Cost_estimate *cost);
    virtual ha_rows multi_range_read_info(uint keyno, uint n_ranges, uint keys,
                                        uint key_parts, uint *bufsz, 
                                        uint *mrr_mode, Cost_estimate *cost);
    virtual int multi_range_read_init(RANGE_SEQ_IF *seq, void *seq_init_param,
                                    uint n_ranges, uint mrr_mode, 
                                    HANDLER_BUFFER *buf);
    virtual int multi_range_read_next(range_id_t *range_info);
    virtual int write_row(uchar *buf __attribute__((unused)))
    virtual int update_row(const uchar *old_data __attribute__((unused)),
    virtual int delete_row(const uchar *buf __attribute__((unused)))
    virtual int reset() { return 0; }
    virtual void start_bulk_insert(ha_rows rows) {}
    virtual int end_bulk_insert() { return 0; }
    virtual int disable_indexes(uint mode) { return HA_ERR_WRONG_COMMAND; }
    virtual int enable_indexes(uint mode) { return HA_ERR_WRONG_COMMAND; }
    virtual int discard_or_import_tablespace(my_bool discard)
    virtual void prepare_for_alter() { return; }
    virtual void drop_table(const char *name);
    virtual int create(const char *name, TABLE *form, HA_CREATE_INFO *info)=0;
    virtual int drop_partitions(const char *path)
    virtual int rename_partitions(const char *path)
    virtual handlerton *partition_ht() const
    ......
}
```

### InnoDB
InnoDB是Mysql最常用的存储引擎，也是默认引擎。结构如下：

![innodb-architecture-5-7.png](https://ping666.com/wp-content/uploads/2024/09/innodb-architecture-5-7.png "innodb-architecture-5-7.png")
InnoDB的几个核心结构包括：B+树，WAL，Undo log，Redo log等。

```cpp
#define ha_innobase ha_innodb //ha_这里是handler abstract的缩写
class ha_innobase: public handler
{
public：
    int write_row(uchar * buf);
}
```
write_row写入接口涉及的核心结构是ins_node_struct
```cpp
struct ins_node_struct{
	que_common_t	common;	/*!< node type: QUE_NODE_INSERT */
	ulint		ins_type;/* INS_VALUES, INS_SEARCHED, or INS_DIRECT */
	dtuple_t*	row;	/*!< row to insert */
	dict_table_t*	table;	/*!< table where to insert */
	sel_node_t*	select;	/*!< select in searched insert */
	que_node_t*	values_list;/* list of expressions to evaluate and
				insert in an INS_VALUES insert */
	ulint		state;	/*!< node execution state */
	dict_index_t*	index;	/*!< NULL, or the next index where the index
				entry should be inserted */
	dtuple_t*	entry;	/*!< NULL, or entry to insert in the index;
				after a successful insert of the entry,
				this should be reset to NULL */
	UT_LIST_BASE_NODE_T(dtuple_t)
			entry_list;/* list of entries, one for each index */
	byte*		row_id_buf;/* buffer for the row id sys field in row */
	trx_id_t	trx_id;	/*!< trx id or the last trx which executed the
				node */
	byte*		trx_id_buf;/* buffer for the trx id sys field in row */
	mem_heap_t*	entry_sys_heap;
				/* memory heap used as auxiliary storage;
				entry_list and sys fields are stored here;
				if this is NULL, entry list should be created
				and buffers for sys fields in row allocated */
	ulint		magic_n;
};
```
write_row的执行流程：

1. thd_to_trx检查是否开启
1. update_auto_increment更细自增字段值
1. innobase_srv_conc_enter_innodb检查线程是否需要等待
1. row_insert_for_mysql插入Innodb表
   1. row_mysql_delay_if_needed检查是否需要等待
   1. trx_start_if_not_started_xa开启事务
   1. row_mysql_convert_row_to_innobasemy转换行数据为innodb格式
   1. trx_savept_take记录事务safepoint
   1. que_fork_get_first_thr获取一个排队任务que_thr_struct
   1. que_thr_move_to_run_state_for_mysql开始执行排队任务
   1. row_ins_step写入表
      1. trx_start_if_not_started_xa开启事务
      1. trx_write_trx_id写一个排他锁IX
      1. lock_table如果需要加表锁LOCK_IX
      1. row_ins开始写数
         1. row_ins_alloc_row_id_step如果没有主键和唯一索引使用row_id做索引
         1. row_ins_index_entry_step为每个索引都写入数据
   1. fts_trx_add_op处理索引
   1. row_update_statistics_if_needed更新一些统计值
1. innobase_srv_conc_exit_innodb解除线程的等待

从row_ins_index_entry_step开始，后面就是B+树的写入操作了row_ins_index_entry_low。
先尝试乐观插入（参数mode=BTR_MODIFY_LEAF），再尝试悲观插入（参数mode=BTR_MODIFY_TREE），执行步骤如下：

1. mtr_start开启mini-transaction
1. btr_cur_search_to_nth_level查找B+树节点位置
   1. mtr_set_savepoint(mtr)保存mini-transaction状态
   1. mtr_x_lock(mtr)/mtr_s_lock(mtr)视情况加写锁或者共享锁
   1. dict_index_get_page获取page页
   1. buf_page_get_gen找到对应的block
   1. page_cur_search_with_match查找记录
1. row_ins_must_modify_rec当前节点位置是否已经存在记录
1. row_ins_clust_index_entry_by_modify如果已经存在记录则修改写入
1. btr_cur_optimistic_insert/btr_cur_pessimistic_insert如果不存在记录直接插入
1. mtr_commit关闭mini-transaction

### B+树

##### B+树的两个特征：
相比普通的B树，B+树有以下两个额外的约束：

- 数据都存在叶子节点，非叶子节点只有key
- 所有叶子节点有序可遍历

![b.png](https://ping666.com/wp-content/uploads/2024/10/b.png "b.png")

##### B+树主要结构：

- 磁盘页page_t
  代码在`storage\innobase\include\page0page.h`，没有一个具体的结构体，全是指针偏移。索引页去头94去尾12，其他空间是可以存索引数据的。
```c
  #define PAGE_DATA	(PAGE_HEADER + 36 + 2 * FSEG_HEADER_SIZE) /* 38 + 36 + 2 * 10 = 94 start of data on the page */
/* The offset of the physically lower end of the directory, counted from
page end, when the page is empty */
#define PAGE_EMPTY_DIR_START	(PAGE_DIR + 2 * PAGE_DIR_SLOT_SIZE) // 8 + 2 * 2 = 12
```
- 记录的逻辑结构dtuple_struct
- 索引的内存结构dict_index_t
- 记录的磁盘结构rec_t
- 索引的游标btr_pcur_t

B+树存储的核心数据结构是磁盘页Page，和Linux虚拟内存的页是类似的作用，也密切相关。比如都有LRU缓存和Page fault等逻辑。其他索引、记录、游标等结构都是基于页的。

Mysql页大小innodb_page_size默认值16K，其他值为4K,8K,32K,64K（对应的X86 Linux内存页大小一般默认为getconf PAGE_SIZE=4K）。常见的表主键字段如雪花数是8个字节，加上索引6个字节。16K的索引页去头去尾一页大概能存1000左右的记录。2层索引页就是1000*1000=100万条，再加上数据页16K/每行记录（比如4K）=4，3层树总共存400万条数据，4层就是40亿条。

为什么说大部分的B+树最好只用3层呢？如下图机械硬盘的寻址时间大概是10ms一次。每多一层需要多一次磁盘寻址。所以用固态硬盘来安装数据库会极大的提升性能。
![computer-time.jpeg](https://ping666.com/wp-content/uploads/2024/11/computer-time.jpeg "computer-time.jpeg")

##### B+树的搜索过程
主要是树游走 + 页内二分查找。
```c
UNIV_INTERN
void
page_cur_search_with_match(
/*=======================*/
	const buf_block_t*	block,	/*!< in: buffer block */
	const dict_index_t*	index,	/*!< in: record descriptor */
	const dtuple_t*		tuple,	/*!< in: data tuple */
	ulint			mode,	/*!< in: PAGE_CUR_L,
					PAGE_CUR_LE, PAGE_CUR_G, or
					PAGE_CUR_GE */
	ulint*			iup_matched_fields,
					/*!< in/out: already matched
					fields in upper limit record */
	ulint*			iup_matched_bytes,
					/*!< in/out: already matched
					bytes in a field not yet
					completely matched */
	ulint*			ilow_matched_fields,
					/*!< in/out: already matched
					fields in lower limit record */
	ulint*			ilow_matched_bytes,
					/*!< in/out: already matched
					bytes in a field not yet
					completely matched */
	page_cur_t*		cursor)	/*!< out: page cursor */
{
	ulint		up;
	ulint		low;
	ulint		mid;
	const page_t*	page;
	const page_dir_slot_t* slot;
	const rec_t*	up_rec;
	const rec_t*	low_rec;
	const rec_t*	mid_rec;
	ulint		up_matched_fields;
	ulint		up_matched_bytes;
	ulint		low_matched_fields;
	ulint		low_matched_bytes;
	ulint		cur_matched_fields;
	ulint		cur_matched_bytes;
	int		cmp;
    ......
	mem_heap_t*	heap		= NULL;
	ulint		offsets_[REC_OFFS_NORMAL_SIZE];
	ulint*		offsets		= offsets_;
	rec_offs_init(offsets_);

	ut_ad(block && tuple && iup_matched_fields && iup_matched_bytes
	      && ilow_matched_fields && ilow_matched_bytes && cursor);
	ut_ad(dtuple_validate(tuple));
    ......
	page_check_dir(page);
    ......
	/* If mode PAGE_CUR_G is specified, we are trying to position the
	cursor to answer a query of the form "tuple < X", where tuple is
	the input parameter, and X denotes an arbitrary physical record on
	the page. We want to position the cursor on the first X which
	satisfies the condition. */

	up_matched_fields  = *iup_matched_fields;
	up_matched_bytes   = *iup_matched_bytes;
	low_matched_fields = *ilow_matched_fields;
	low_matched_bytes  = *ilow_matched_bytes;

	/* Perform binary search. First the search is done through the page
	directory, after that as a linear search in the list of records
	owned by the upper limit directory slot. */

	low = 0;
	up = page_dir_get_n_slots(page) - 1;

	/* Perform binary search until the lower and upper limit directory
	slots come to the distance 1 of each other */

	while (up - low > 1) {
		mid = (low + up) / 2;
		slot = page_dir_get_nth_slot(page, mid);
		mid_rec = page_dir_slot_get_rec(slot);

		ut_pair_min(&cur_matched_fields, &cur_matched_bytes,
			    low_matched_fields, low_matched_bytes,
			    up_matched_fields, up_matched_bytes);

		offsets = rec_get_offsets(mid_rec, index, offsets,
					  dtuple_get_n_fields_cmp(tuple),
					  &heap);

		cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, offsets,
						&cur_matched_fields,
						&cur_matched_bytes);
		if (UNIV_LIKELY(cmp > 0)) {
low_slot_match:
			low = mid;
			low_matched_fields = cur_matched_fields;
			low_matched_bytes = cur_matched_bytes;

		} else if (UNIV_EXPECT(cmp, -1)) {
        ......
up_slot_match:
			up = mid;
			up_matched_fields = cur_matched_fields;
			up_matched_bytes = cur_matched_bytes;

		} else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE
#ifdef PAGE_CUR_LE_OR_EXTENDS
			   || mode == PAGE_CUR_LE_OR_EXTENDS
#endif /* PAGE_CUR_LE_OR_EXTENDS */
			   ) {

			goto low_slot_match;
		} else {

			goto up_slot_match;
		}
	}

	slot = page_dir_get_nth_slot(page, low);
	low_rec = page_dir_slot_get_rec(slot);
	slot = page_dir_get_nth_slot(page, up);
	up_rec = page_dir_slot_get_rec(slot);

	/* Perform linear search until the upper and lower records come to
	distance 1 of each other. */

	while (page_rec_get_next_const(low_rec) != up_rec) {

		mid_rec = page_rec_get_next_const(low_rec);

		ut_pair_min(&cur_matched_fields, &cur_matched_bytes,
			    low_matched_fields, low_matched_bytes,
			    up_matched_fields, up_matched_bytes);

		offsets = rec_get_offsets(mid_rec, index, offsets,
					  dtuple_get_n_fields_cmp(tuple),
					  &heap);

		cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, offsets,
						&cur_matched_fields,
						&cur_matched_bytes);
		if (UNIV_LIKELY(cmp > 0)) {
low_rec_match:
			low_rec = mid_rec;
			low_matched_fields = cur_matched_fields;
			low_matched_bytes = cur_matched_bytes;

		} else if (UNIV_EXPECT(cmp, -1)) {
#ifdef PAGE_CUR_LE_OR_EXTENDS
			if (mode == PAGE_CUR_LE_OR_EXTENDS
			    && page_cur_rec_field_extends(
				    tuple, mid_rec, offsets,
				    cur_matched_fields)) {

				goto low_rec_match;
			}
#endif /* PAGE_CUR_LE_OR_EXTENDS */
up_rec_match:
			up_rec = mid_rec;
			up_matched_fields = cur_matched_fields;
			up_matched_bytes = cur_matched_bytes;
		} else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE
#ifdef PAGE_CUR_LE_OR_EXTENDS
			   || mode == PAGE_CUR_LE_OR_EXTENDS
#endif /* PAGE_CUR_LE_OR_EXTENDS */
			   ) {

			goto low_rec_match;
		} else {

			goto up_rec_match;
		}
	}
    ......
	if (mode <= PAGE_CUR_GE) {
		page_cur_position(up_rec, block, cursor);
	} else {
		page_cur_position(low_rec, block, cursor);
	}

	*iup_matched_fields  = up_matched_fields;
	*iup_matched_bytes   = up_matched_bytes;
	*ilow_matched_fields = low_matched_fields;
	*ilow_matched_bytes  = low_matched_bytes;
	if (UNIV_LIKELY_NULL(heap)) {
		mem_heap_free(heap);
	}
}  
```

##### B+树的写分裂
在btr_cur_optimistic_insert写数据的过程中，会遇到页记录满了写不下的情况，这时需要分裂页。

页的分裂是很耗时的，需要把一部分记录挪到新的页上去，有时还会触发多层的页分裂。所以一般主键最好是有递增趋势的，这样每次记录都是顺序写在后面。如果主键值是随机的，就会频繁触发分裂影响性能。内置的自增字段auto_increment，在分库分表时可能会遇到一些麻烦。一般使用雪花id代替。雪花id是一种UUID的变体。时间戳 + 机器硬件序列号 + 本地序列，将时间戳放到数值前面，以达到趋势递增的目的。

```c
UNIV_INTERN
rec_t*
btr_page_split_and_insert(
/*======================*/
	btr_cur_t*	cursor,	/*!< in: cursor at which to insert; when the
				function returns, the cursor is positioned
				on the predecessor of the inserted record */
	const dtuple_t*	tuple,	/*!< in: tuple to insert */
	ulint		n_ext,	/*!< in: number of externally stored columns */
	mtr_t*		mtr)	/*!< in: mtr */
{
	buf_block_t*	block;
	page_t*		page;
	page_zip_des_t*	page_zip;
	ulint		page_no;
	byte		direction;
	ulint		hint_page_no;
	buf_block_t*	new_block;
	page_t*		new_page;
	page_zip_des_t*	new_page_zip;
	rec_t*		split_rec;
	buf_block_t*	left_block;
	buf_block_t*	right_block;
	buf_block_t*	insert_block;
	page_cur_t*	page_cursor;
	rec_t*		first_rec;
	byte*		buf = 0; /* remove warning */
	rec_t*		move_limit;
	ibool		insert_will_fit;
	ibool		insert_left;
	ulint		n_iterations = 0;
	rec_t*		rec;
	mem_heap_t*	heap;
	ulint		n_uniq;
	ulint*		offsets;

	heap = mem_heap_create(1024);
	n_uniq = dict_index_get_n_unique_in_tree(cursor->index);
func_start:
	mem_heap_empty(heap);
	offsets = NULL;

	ut_ad(mtr_memo_contains(mtr, dict_index_get_lock(cursor->index),
				MTR_MEMO_X_LOCK));
#ifdef UNIV_SYNC_DEBUG
	ut_ad(rw_lock_own(dict_index_get_lock(cursor->index), RW_LOCK_EX));
#endif /* UNIV_SYNC_DEBUG */

	block = btr_cur_get_block(cursor);
	page = buf_block_get_frame(block);
	page_zip = buf_block_get_page_zip(block);

	ut_ad(mtr_memo_contains(mtr, block, MTR_MEMO_PAGE_X_FIX));
	ut_ad(page_get_n_recs(page) >= 1);

	page_no = buf_block_get_page_no(block);

	/* 1. Decide the split record; split_rec == NULL means that the
	tuple to be inserted should be the first record on the upper
	half-page */
	insert_left = FALSE;

	if (n_iterations > 0) {
		direction = FSP_UP;
		hint_page_no = page_no + 1;
		split_rec = btr_page_get_split_rec(cursor, tuple, n_ext);

		if (split_rec == NULL) {
			insert_left = btr_page_tuple_smaller(
				cursor, tuple, offsets, n_uniq, &heap);
		}
	} else if (btr_page_get_split_rec_to_right(cursor, &split_rec)) {
		direction = FSP_UP;
		hint_page_no = page_no + 1;

	} else if (btr_page_get_split_rec_to_left(cursor, &split_rec)) {
		direction = FSP_DOWN;
		hint_page_no = page_no - 1;
		ut_ad(split_rec);
	} else {
		direction = FSP_UP;
		hint_page_no = page_no + 1;

		/* If there is only one record in the index page, we
		can't split the node in the middle by default. We need
		to determine whether the new record will be inserted
		to the left or right. */

		if (page_get_n_recs(page) > 1) {
			split_rec = page_get_middle_rec(page);
		} else if (btr_page_tuple_smaller(cursor, tuple,
						  offsets, n_uniq, &heap)) {
			split_rec = page_rec_get_next(
				page_get_infimum_rec(page));
		} else {
			split_rec = NULL;
		}
	}

	/* 2. Allocate a new page to the index */
	new_block = btr_page_alloc(cursor->index, hint_page_no, direction,
				   btr_page_get_level(page, mtr), mtr, mtr);
	new_page = buf_block_get_frame(new_block);
	new_page_zip = buf_block_get_page_zip(new_block);
	btr_page_create(new_block, new_page_zip, cursor->index,
			btr_page_get_level(page, mtr), mtr);

	/* 3. Calculate the first record on the upper half-page, and the
	first record (move_limit) on original page which ends up on the
	upper half */

	if (split_rec) {
		first_rec = move_limit = split_rec;

		offsets = rec_get_offsets(split_rec, cursor->index, offsets,
					  n_uniq, &heap);

		insert_left = cmp_dtuple_rec(tuple, split_rec, offsets) < 0;

		if (!insert_left && new_page_zip && n_iterations > 0) {
			/* If a compressed page has already been split,
			avoid further splits by inserting the record
			to an empty page. */
			split_rec = NULL;
			goto insert_empty;
		}
	} else if (insert_left) {
		ut_a(n_iterations > 0);
		first_rec = page_rec_get_next(page_get_infimum_rec(page));
		move_limit = page_rec_get_next(btr_cur_get_rec(cursor));
	} else {
insert_empty:
		ut_ad(!split_rec);
		ut_ad(!insert_left);
		buf = (byte*) mem_alloc(rec_get_converted_size(cursor->index,
							       tuple, n_ext));

		first_rec = rec_convert_dtuple_to_rec(buf, cursor->index,
						      tuple, n_ext);
		move_limit = page_rec_get_next(btr_cur_get_rec(cursor));
	}

	/* 4. Do first the modifications in the tree structure */

	btr_attach_half_pages(cursor->index, block,
			      first_rec, new_block, direction, mtr);

	/* If the split is made on the leaf level and the insert will fit
	on the appropriate half-page, we may release the tree x-latch.
	We can then move the records after releasing the tree latch,
	thus reducing the tree latch contention. */

	if (split_rec) {
		insert_will_fit = !new_page_zip
			&& btr_page_insert_fits(cursor, split_rec,
						offsets, tuple, n_ext, heap);
	} else {
		if (!insert_left) {
			mem_free(buf);
			buf = NULL;
		}

		insert_will_fit = !new_page_zip
			&& btr_page_insert_fits(cursor, NULL,
						NULL, tuple, n_ext, heap);
	}

	if (insert_will_fit && page_is_leaf(page)) {

		mtr_memo_release(mtr, dict_index_get_lock(cursor->index),
				 MTR_MEMO_X_LOCK);
	}

	/* 5. Move then the records to the new page */
	if (direction == FSP_DOWN) {
		/*		fputs("Split left\n", stderr); */

		if (0
#ifdef UNIV_ZIP_COPY
		    || page_zip
#endif /* UNIV_ZIP_COPY */
		    || !page_move_rec_list_start(new_block, block, move_limit,
					       cursor->index, mtr)) {
			/* For some reason, compressing new_page failed,
			even though it should contain fewer records than
			the original page.  Copy the page byte for byte
			and then delete the records from both pages
			as appropriate.  Deleting will always succeed. */
			ut_a(new_page_zip);

			page_zip_copy_recs(new_page_zip, new_page,
					   page_zip, page, cursor->index, mtr);
			page_delete_rec_list_end(move_limit - page + new_page,
						 new_block, cursor->index,
						 ULINT_UNDEFINED,
						 ULINT_UNDEFINED, mtr);

			/* Update the lock table and possible hash index. */

			lock_move_rec_list_start(
				new_block, block, move_limit,
				new_page + PAGE_NEW_INFIMUM);

			btr_search_move_or_delete_hash_entries(
				new_block, block, cursor->index);

			/* Delete the records from the source page. */

			page_delete_rec_list_start(move_limit, block,
						   cursor->index, mtr);
		}

		left_block = new_block;
		right_block = block;

		lock_update_split_left(right_block, left_block);
	} else {
		/*		fputs("Split right\n", stderr); */

		if (0
#ifdef UNIV_ZIP_COPY
		    || page_zip
#endif /* UNIV_ZIP_COPY */
		    || !page_move_rec_list_end(new_block, block, move_limit,
					     cursor->index, mtr)) {
			/* For some reason, compressing new_page failed,
			even though it should contain fewer records than
			the original page.  Copy the page byte for byte
			and then delete the records from both pages
			as appropriate.  Deleting will always succeed. */
			ut_a(new_page_zip);

			page_zip_copy_recs(new_page_zip, new_page,
					   page_zip, page, cursor->index, mtr);
			page_delete_rec_list_start(move_limit - page
						   + new_page, new_block,
						   cursor->index, mtr);

			/* Update the lock table and possible hash index. */

			lock_move_rec_list_end(new_block, block, move_limit);

			btr_search_move_or_delete_hash_entries(
				new_block, block, cursor->index);

			/* Delete the records from the source page. */

			page_delete_rec_list_end(move_limit, block,
						 cursor->index,
						 ULINT_UNDEFINED,
						 ULINT_UNDEFINED, mtr);
		}

		left_block = block;
		right_block = new_block;

		lock_update_split_right(right_block, left_block);
	}

#ifdef UNIV_ZIP_DEBUG
	if (page_zip) {
		ut_a(page_zip_validate(page_zip, page));
		ut_a(page_zip_validate(new_page_zip, new_page));
	}
#endif /* UNIV_ZIP_DEBUG */

	/* At this point, split_rec, move_limit and first_rec may point
	to garbage on the old page. */

	/* 6. The split and the tree modification is now completed. Decide the
	page where the tuple should be inserted */

	if (insert_left) {
		insert_block = left_block;
	} else {
		insert_block = right_block;
	}

	/* 7. Reposition the cursor for insert and try insertion */
	page_cursor = btr_cur_get_page_cur(cursor);

	page_cur_search(insert_block, cursor->index, tuple,
			PAGE_CUR_LE, page_cursor);

	rec = page_cur_tuple_insert(page_cursor, tuple,
				    cursor->index, n_ext, mtr);

#ifdef UNIV_ZIP_DEBUG
	{
		page_t*		insert_page
			= buf_block_get_frame(insert_block);

		page_zip_des_t*	insert_page_zip
			= buf_block_get_page_zip(insert_block);

		ut_a(!insert_page_zip
		     || page_zip_validate(insert_page_zip, insert_page));
	}
#endif /* UNIV_ZIP_DEBUG */

	if (rec != NULL) {

		goto func_exit;
	}

	/* 8. If insert did not fit, try page reorganization */

	if (!btr_page_reorganize(insert_block, cursor->index, mtr)) {

		goto insert_failed;
	}

	page_cur_search(insert_block, cursor->index, tuple,
			PAGE_CUR_LE, page_cursor);
	rec = page_cur_tuple_insert(page_cursor, tuple, cursor->index,
				    n_ext, mtr);

	if (rec == NULL) {
		/* The insert did not fit on the page: loop back to the
		start of the function for a new split */
insert_failed:
		/* We play safe and reset the free bits for new_page */
		if (!dict_index_is_clust(cursor->index)) {
			ibuf_reset_free_bits(new_block);
		}

		/* fprintf(stderr, "Split second round %lu\n",
		page_get_page_no(page)); */
		n_iterations++;
		ut_ad(n_iterations < 2
		      || buf_block_get_page_zip(insert_block));
		ut_ad(!insert_will_fit);

		goto func_start;
	}

func_exit:
	/* Insert fit on the page: update the free bits for the
	left and right pages in the same mtr */

	if (!dict_index_is_clust(cursor->index) && page_is_leaf(page)) {
		ibuf_update_free_bits_for_two_pages_low(
			buf_block_get_zip_size(left_block),
			left_block, right_block, mtr);
	}

#if 0
	fprintf(stderr, "Split and insert done %lu %lu\n",
		buf_block_get_page_no(left_block),
		buf_block_get_page_no(right_block));
#endif
	MONITOR_INC(MONITOR_INDEX_SPLIT);

	ut_ad(page_validate(buf_block_get_frame(left_block), cursor->index));
	ut_ad(page_validate(buf_block_get_frame(right_block), cursor->index));

	mem_heap_free(heap);
	return(rec);
}
```

### 事务ACID
描述事务（Transaction）的四个维度：

- 原子性（Atomicity）
  支持原子性的3个命令begin、commit、rollback。当用户执行rollback时，采用undo log来恢复数据。
- 一致性（Consistency）
  是指事务结束前后，数据库都处于一个合法的状态：主键、外键、索引、字段等约束都要满足要求。
- 隔离性（Isolation）
  并行的事务之间是数据是如何互相影响的。SQL规范定义了4种隔离级别。
- 持久性（Durability）
  因为性能要求，不可能每个写操作都直接刷磁盘。当进程意外退出时，需要采用redo log来恢复数据。Innodb的redo log采用WAL（Write-ahead logging）来保证同时刷内存和磁盘。

### 事务隔离
SQL99协议标准在这里[https://web.cecs.pdx.edu/~len/sql1999.pdf](https://web.cecs.pdx.edu/~len/sql1999.pdf "https://web.cecs.pdx.edu/~len/sql1999.pdf")

- SQL99标准定义的事务隔离级别
![sql-transaction-isolation.png](https://ping666.com/wp-content/uploads/2024/09/sql-transaction-isolation.png "sql-transaction-isolation.png")
第一个没有隔离，第四个没有并发，这两个实际场景中都很少使用。Oracle默认是READ-COMMITTED，Innodb的默认事务隔离级别是REPEATABLE-READ，采用MVCC实现。

- 查看数据库事务设置
  通过以下命令可以查看数据库当前事务和锁情况
```sql
select @@tx_isolation, @@autocommit;
select * from information_schema.innodb_trx;
select * from information_schema.innodb_locks;
select * from information_schema.innodb_lock_waits;
```

### MVCC
InnoDB使用Multi-Version Concurrency Control（多版本并发控制协议）来解决脏读、不可重复度。和Gap锁配合可以避免部分幻读。
![mysql-mvcc.png](https://ping666.com/wp-content/uploads/2024/10/mysql-mvcc.png "mysql-mvcc.png")

每条记录都有2个隐含的字段：trx_id（事务ID）和roll_ptr（对应的Undo log）。每当事务开启时会创建read_view对象：记录了当前事务列表的最小、最大事务ID和事务ID列表。当前事务的读操作全部通过read_view完成，以此来实现多版本控制，达到一致性读的目标。
read_view的代码在`storage\innobase\include\read0read.h`
```cpp
/** Read view lists the trx ids of those transactions for which a consistent
read should not see the modifications to the database. */

struct read_view_struct{
	ulint		type;	/*!< VIEW_NORMAL, VIEW_HIGH_GRANULARITY */
	undo_no_t	undo_no;/*!< 0 or if type is
				VIEW_HIGH_GRANULARITY
				transaction undo_no when this high-granularity
				consistent read view was created */
	trx_id_t	low_limit_no;
				/*!< The view does not need to see the undo
				logs for transactions whose transaction number
				is strictly smaller (<) than this value: they
				can be removed in purge if not needed by other
				views */
	trx_id_t	low_limit_id;
				/*!< The read should not see any transaction
				with trx id >= this value. In other words,
				this is the "high water mark". */
	trx_id_t	up_limit_id;
				/*!< The read should see all trx ids which
				are strictly smaller (<) than this value.
				In other words,
				this is the "low water mark". */
	ulint		n_trx_ids;
				/*!< Number of cells in the trx_ids array */
	trx_id_t*	trx_ids;/*!< Additional trx ids which the read should
				not see: typically, these are the read-write
				active transactions at the time when the read
			       	is serialized, except the reading transaction
				itself; the trx ids in this array are in a
				descending order. These trx_ids should be
				between the "low" and "high" water marks,
				that is, up_limit_id and low_limit_id. */
	trx_id_t	creator_trx_id;
				/*!< trx id of creating transaction, or
				0 used in purge */
	UT_LIST_NODE_T(read_view_t) view_list;
				/*!< List of read views in trx_sys */
};
```

### 高可用

- 涉及数据安全的几个关键参数
  - innodb_flush_log_at_trx_commit（redo log持久化策略）
     - =0，redo log只写内存，由master thread每秒钟定时刷磁盘。性能最好，但是数据库异常时会丢失1秒以内的数据。
     - =1，每次事务提交后写磁盘，并且执行fsync。默认值，性能最差，数据库异常时最多丢失1个事务的数据。
     - =2，每次事务后，redo log被写入文件系统的缓存。安全性和性能介于0和1之间。
  - sync_binlog（=1开启binlog，需要手动开启）
  - 定时备份数据库（全量mysqldump + 增量binlog）
- 主从复制
  从节点通过tcp连接到主节点，拿到binlog后在从节点上重放以同步数据。这种复制从节点上的数据会有一点的延迟。对实时性要求不高的读操作可以访问这里。查看binlog功能的状态：
```sql
show variables like '%log_bin%';
show master status;
```
- NDB Cluster
  作为官方提供的分布式集群方案，依赖NDB存储引擎（和Innodb引擎差异比较大）。换存储引擎相当于换数据库了，不如一步到位换新一代的数据库架构，比如Tidb。
![mysql-ndb-cluster.png](https://ping666.com/wp-content/uploads/2024/10/mysql-ndb-cluster.png "mysql-ndb-cluster.png")