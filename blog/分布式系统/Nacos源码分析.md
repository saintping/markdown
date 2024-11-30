### 前言
Nacos，Dynamic Naming and Configuration Service。是阿里开源的一款配置中心服务，使用Java实现。[https://github.com/alibaba/nacos](https://github.com/alibaba/nacos "https://github.com/alibaba/nacos")

与其旗鼓相当的是以Go实现的Consul，在Github上都有30k左右的Star。在云原生微服务兴起之前，分布式服务领域曾经的主流配置中心是Zookeeper。然后是NetFlix开源的Eureka，作为Spring Boot的默认方案一时也有非常广泛的使用。

以前一直是Zookeeper用的比较多，这次看一下Nacos。以下是几个主要配置中心的对比。

| 对比项目 | Zookeeper | Eureka | Consul | Nacos |
| :------------ | :------------ | :------------ | :------------ | :------------ |
| CAP | CP | AP | CP | CP/AP(只支持临时实例，源码中有接口没看到实现) |
| 访问协议 | TCP | HTTP | HTTP/DNS | HTTP/DNS |
| 多数据中心 | 不支持 | 支持 | 支持 | 支持（Nacos-Sync） |
| 跨中心同步 | 不支持 | 不支持 | 支持 | 支持（Nacos-Sync） |
| 雪崩保护 | 无 | 有 | 无 | 有 |
| K8s集成 | 不支持 | 不支持 | 支持 | 支持 |

### Nacos架构

![nacos-arch.jpeg](https://ping666.com/wp-content/uploads/2024/11/nacos-arch.jpeg "nacos-arch.jpeg")

- 命名服务
  服务注册、服务状态检测、服务发现（除了HTTP，也支持DNS协议），数据通过一致性协议JRaft写本地磁盘RocksDB。
- 配置中心
  配置管理、被动读取配置、主动下发配置，支持将数据写在Derby（单机模式）或者外接存储中Mysql（集群模式）。

### 集群部署
这里使用单机部署一个伪集群，只为体验Naming中的一致性协议JRaft。

- 创建外部数据库
集群模式需要外接数据库，需要手动创建表结构`config\src\main\resources\META-INF\mysql-schema.sql`。并且配置到application.properties中。
```asp
spring.sql.init.platform=mysql
### Count of DB:
db.num=1
### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
db.user.0=root
db.password.0=123456
```

- 启动多实例
  application.properties：各种API默认使用8848端口，其他2个实例依次为8858和8868
  cluster.conf：每行一个实例如10.39.163.51:8848。
  startup.sh启动进程
  `INFO Nacos started successfully in cluster mode. use external storage`
![nacos-admin.png](https://ping666.com/wp-content/uploads/2024/11/nacos-admin.png "nacos-admin.png")

**注意**

- gRPC客户端的端口在默认端口上对应+1000即9848，gRPC服务端的端口对应+1001即9849，JRaft端口对应-1000即7848。
- 会删除状态异常的临时实例，同时会定期清理没有实例的空服务。nacos.naming.clean.empty-service.interval=60000
- 如果机器是多网卡的，可以设置nacos.inetutils.ip-address=10.39.163.51
- 集群重启异常&选举失败，可以先直接删除data下的文件（生产环境涉慎用）。

### 客户端代码
Java客户端
`implementation 'com.alibaba.nacos:nacos-client:2.4.0'`

- 命名服务
```java
NamingService naming = NamingFactory.createNamingService(System.getProperty("serveAddr"));
// 以下注册请求所造成的结果均一致, 注册分组名为`DEFAULT_GROUP`, 服务名为`nacos.test.service`的实例，实例的ip为`127.0.0.1`, port为`8848`, clusterName为`DEFAULT`.
Instance instance = new Instance();
instance.setIp("127.0.0.1");
instance.setPort(8848);
instance.setClusterName("DEFAULT");
naming.registerInstance("nacos.test.service", "DEFAULT_GROUP", instance);
// 注销服务
naming.deregisterInstance("nacos.test.service", "DEFAULT_GROUP", instance);
// 查询服务列表
List<Instance> instances = naming.getAllInstances(String serviceName);
```
和Spring更简单的集成，注解`@EnableDiscoveryClient`
`implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-discovery:2.2.10.RELEASE'`

- 配置中心
```java
String serverAddr = "{serverAddr}";
String dataId = "{dataId}";
String group = "{group}";
Properties properties = new Properties();
properties.put("serverAddr", serverAddr);
ConfigService configService = NacosFactory.createConfigService(properties);
// 服务启动时查询配置
String content = configService.getConfig(dataId, group, 5000);
System.out.println(content);
// 服务运行中监听动态下发的配置
configService.addListener(dataId, group, new Listener() {
    @Override
    public void receiveConfigInfo(String configInfo) {
        System.out.println("recieve1:" + configInfo);
    }
    @Override
    public Executor getExecutor() {
        return null;
    }
});
```
和Spring更简单的集成，支持Spring原生注解@Value和@RefreshScope
`implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-config:2.2.10.RELEASE'`

也可以通过API直接管理配置，不过一般直接从管理端配置。

### 服务端源码

- 启动入口
  `start.sh`脚本启动的进程实际是`-jar /data/nacos/target/nacos-server.jar`
```java
@SpringBootApplication
//Spring注解加载com.alibaba.nacos包里的所有@Component（console、naming、config、core这4个包里有很多@Controller/@Service），默认只加载@Bean
@ComponentScan(basePackages = "com.alibaba.nacos", excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = {NacosTypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {TypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {AutoConfigurationExcludeFilter.class})})
@ServletComponentScan
//有很多@Scheduled注解的定时任务，Spring注解加载过程参见ScheduledAnnotationBeanPostProcessor#postProcessAfterInitialization
@EnableScheduling
public class Nacos {
    public static void main(String[] args) {
        SpringApplication.run(Nacos.class, args);
    }
}
```

- nacos-console
  服务、模块的状态接口，以及名字空间的CRUD。默认的名字空间是public，在DEFAULT_GROUP组下。
```asp
/nacos/v2/console/health/**
/nacos/v1/console/server/**
/nacos/v2/console/namespaces/**
```

- nacos-config
```asp
/nacos/v2/cs/config/**
/nacos/v2/cs/history/**
```
ConfigControllerV2/HistoryControllerV2的API都会调用ConfigInfoPersistService存储接口，这个接口有内置和外接两种实现。其中外接的存储实现ExternalConfigInfoPersistServiceImpl还做了插件化MapperManager，然后通过JdbcTemplate读写。
```java
public class ExternalConfigInfoPersistServiceImpl implements ConfigInfoPersistService {    
    public long addConfigInfoAtomic(final long configId, final String srcIp, final String srcUser,
            final ConfigInfo configInfo, Map<String, Object> configAdvanceInfo) {
        
        KeyHolder keyHolder = ExternalStorageUtils.createKeyHolder();
        
        ConfigInfoMapper configInfoMapper = mapperManager.findM
			apper(dataSourceService.getDataSourceType(),
                TableConstant.CONFIG_INFO);
        try {
            jt.update(
                    connection -> createPsForInsertConfigInfo(srcIp, srcUser, configInfo, configAdvanceInfo, connection,
                            configInfoMapper), keyHolder);
            Number nu = keyHolder.getKey();
            if (nu == null) {
                throw new IllegalArgumentException("insert config_info fail");
            }
            return nu.longValue();
        } catch (CannotGetJdbcConnectionException e) {
            LogUtil.FATAL_LOG.error("[db-error] " + e, e);
            throw e;
        }
    }
}
```

- nacos-naming
```asp
/nacos/v2/ns/catalog/**
/nacos/v2/ns/client/**
/nacos/v2/ns/health/**
/nacos/v2/ns/instance/**
/nacos/v2/ns/ops/**
/nacos/v2/ns/service/**
```
ServiceControllerV2/InstanceControllerV2等API最终都会调用ConsistencyProtocol接口去写数据。
![ConsistencyProtocol.png](https://ping666.com/wp-content/uploads/2024/11/ConsistencyProtocol.png "ConsistencyProtocol.png")

- nacos-consistency
  Response ConsistencyProtocol.write(WriteRequest request)会调用JRaftServer.commit写数据。如果是不是Leader节点，转发消息，如果是Leader执行node.apply(task)。
```java
public class JRaftServer {
    public CompletableFuture<Response> commit(final String group, final Message data,
            final CompletableFuture<Response> future) {
        LoggerUtils.printIfDebugEnabled(Loggers.RAFT, "data requested this time : {}", data);
        final RaftGroupTuple tuple = findTupleByGroup(group);
        if (tuple == null) {
            future.completeExceptionally(new IllegalArgumentException("No corresponding Raft Group found : " + group));
            return future;
        }
        FailoverClosureImpl closure = new FailoverClosureImpl(future);
        final Node node = tuple.node;
        if (node.isLeader()) {
            // The leader node directly applies this request
            applyOperation(node, data, closure);
        } else {
            // Forward to Leader for request processing
            invokeToLeader(group, data, rpcRequestTimeoutMs, closure);
        }
        return future;
    }
    public void applyOperation(Node node, Message data, FailoverClosure closure) {
        final Task task = new Task();
        task.setDone(new NacosClosure(data, status -> {
            NacosClosure.NacosStatus nacosStatus = (NacosClosure.NacosStatus) status;
            closure.setThrowable(nacosStatus.getThrowable());
            closure.setResponse(nacosStatus.getResponse());
            closure.run(nacosStatus);
        }));
        // add request type field at the head of task data.
        byte[] requestTypeFieldBytes = new byte[2];
        requestTypeFieldBytes[0] = ProtoMessageUtil.REQUEST_TYPE_FIELD_TAG;
        if (data instanceof ReadRequest) {
            requestTypeFieldBytes[1] = ProtoMessageUtil.REQUEST_TYPE_READ;
        } else {
            requestTypeFieldBytes[1] = ProtoMessageUtil.REQUEST_TYPE_WRITE;
        }
        byte[] dataBytes = data.toByteArray();
        task.setData((ByteBuffer) ByteBuffer.allocate(requestTypeFieldBytes.length + dataBytes.length)
                .put(requestTypeFieldBytes).put(dataBytes).position(0));
        node.apply(task);
    }
}
```

- nacos-core
```asp
/nacos/v2/core/ops/**
/nacos/v2/core/cluster/**
```

### Raft实现源码
Nacos的Raft协议，使用的是[https://github.com/sofastack/sofa-jraft](https://github.com/sofastack/sofa-jraft "https://github.com/sofastack/sofa-jraft")，支持MULTI-RAFT-GROUP。文档地址[https://www.sofastack.tech/projects/sofa-jraft/overview](https://www.sofastack.tech/projects/sofa-jraft/overview "https://www.sofastack.tech/projects/sofa-jraft/overview")

以下是系统架构图
![jraft.png](https://ping666.com/wp-content/uploads/2024/11/jraft.png "jraft.png")

- 提交任务Task
  提交任务只是把Task放到Disruptor队列。
```java
public class NodeImpl implements Node, RaftServerService {
    public void apply(final Task task) {
        if (this.shutdownLatch != null) {
            ThreadPoolsFactory.runClosureInThread(this.groupId, task.getDone(), new Status(RaftError.ENODESHUTDOWN, "Node is shutting down."));
            throw new IllegalStateException("Node is shutting down");
        }
        Requires.requireNonNull(task, "Null task");

        final LogEntry entry = new LogEntry();
        entry.setData(task.getData());

        final EventTranslator<LogEntryAndClosure> translator = (event, sequence) -> {
          event.reset();
          event.done = task.getDone();
          event.entry = entry;
          event.expectedTerm = task.getExpectedTerm();
        };

        switch(this.options.getApplyTaskMode()) {
          case Blocking:
            this.applyQueue.publishEvent(translator);
            break;
          case NonBlocking:
          default:
            if (!this.applyQueue.tryPublishEvent(translator)) {
              String errorMsg = "Node is busy, has too many tasks, queue is full and bufferSize="+ this.applyQueue.getBufferSize();
                ThreadPoolsFactory.runClosureInThread(this.groupId, task.getDone(),
                  new Status(RaftError.EBUSY, errorMsg));
              LOG.warn("Node {} applyQueue is overload.", getNodeId());
              this.metrics.recordTimes("apply-task-overload-times", 1);
              if(task.getDone() == null) {
                throw new OverloadException(errorMsg);
              }
            }
            break;
        }
    }
}
```

- 批量处理Task
  Disruptor队列事件句柄LogEntryAndClosureHandler，最多32个Task做1次批处理。
```java
private void executeApplyingTasks(final List<LogEntryAndClosure> tasks) {
    this.writeLock.lock();
    try {
        final int size = tasks.size();
        if (this.state != State.STATE_LEADER) {
            final Status st = new Status();
            if (this.state != State.STATE_TRANSFERRING) {
                st.setError(RaftError.EPERM, "Is not leader.");
            } else {
                st.setError(RaftError.EBUSY, "Is transferring leadership.");
            }
            LOG.debug("Node {} can't apply, status={}.", getNodeId(), st);
            final List<Closure> dones = tasks.stream().map(ele -> ele.done)
                    .filter(Objects::nonNull).collect(Collectors.toList());
            ThreadPoolsFactory.runInThread(this.groupId, () -> {
                for (final Closure done : dones) {
                    done.run(st);
                }
            });
            return;
        }
        final List<LogEntry> entries = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            final LogEntryAndClosure task = tasks.get(i);
            if (task.expectedTerm != -1 && task.expectedTerm != this.currTerm) {
                LOG.debug("Node {} can't apply task whose expectedTerm={} doesn't match currTerm={}.", getNodeId(),
                    task.expectedTerm, this.currTerm);
                if (task.done != null) {
                    final Status st = new Status(RaftError.EPERM, "expected_term=%d doesn't match current_term=%d",
                        task.expectedTerm, this.currTerm);
                    ThreadPoolsFactory.runClosureInThread(this.groupId, task.done, st);
                    task.reset();
                }
                continue;
            }
            if (!this.ballotBox.appendPendingTask(this.conf.getConf(),
                this.conf.isStable() ? null : this.conf.getOldConf(), task.done)) {
                ThreadPoolsFactory.runClosureInThread(this.groupId, task.done, new Status(RaftError.EINTERNAL, "Fail to append task."));
                task.reset();
                continue;
            }
            // set task entry info before adding to list.
            task.entry.getId().setTerm(this.currTerm);
            task.entry.setType(EnumOutter.EntryType.ENTRY_TYPE_DATA);
            entries.add(task.entry);
            task.reset();
        }
        this.logManager.appendEntries(entries, new LeaderStableClosure(entries));
        // update conf.first
        checkAndSetConfiguration(true);
    } finally {
        this.writeLock.unlock();
    }
}
```

- 提交投票
```java
public class BallotBox implements Lifecycle<BallotBoxOptions>, Describer {
    public boolean appendPendingTask(final Configuration conf, final Configuration oldConf, final Closure done) {
        final Ballot bl = new Ballot();
        if (!bl.init(conf, oldConf)) {
            LOG.error("Fail to init ballot.");
            return false;
        }
        final long stamp = this.stampedLock.writeLock();
        try {
            if (this.pendingIndex <= 0) {
                LOG.error("Node {} fail to appendingTask, pendingIndex={}.", this.opts.getNodeId(), this.pendingIndex);
                return false;
            }
            this.pendingMetaQueue.add(bl);
            this.closureQueue.appendPendingClosure(done);
            return true;
        } finally {
            this.stampedLock.unlockWrite(stamp);
        }
    }
}
```

- 计票结果
```java
class LeaderStableClosure extends LogManager.StableClosure {

    public LeaderStableClosure(final List<LogEntry> entries) {
        super(entries);
    }

    @Override
    public void run(final Status status) {
        if (status.isOk()) {
            NodeImpl.this.ballotBox.commitAt(this.firstLogIndex, this.firstLogIndex + this.nEntries - 1,
                NodeImpl.this.serverId);
        } else {
            LOG.error("Node {} append [{}, {}] failed, status={}.", getNodeId(), this.firstLogIndex,
                this.firstLogIndex + this.nEntries - 1, status);
        }
    }
}
```

- LogManager写WAL日志
  将LogEntry写到内存，并且放到磁盘写队列Disruptor。
```java
public class LogManagerImpl implements LogManager {
    public void appendEntries(final List<LogEntry> entries, final StableClosure done) {
        assert(done != null);

        Requires.requireNonNull(done, "done");
        if (this.hasError) {
            entries.clear();
            ThreadPoolsFactory.runClosureInThread(this.groupId, done, new Status(RaftError.EIO, "Corrupted LogStorage"));
            return;
        }
        boolean doUnlock = true;
        this.writeLock.lock();
        try {
            if (!entries.isEmpty() && !checkAndResolveConflict(entries, done, this.writeLock)) {
                // If checkAndResolveConflict returns false, the done will be called in it.
                entries.clear();
                return;
            }
            for (int i = 0; i < entries.size(); i++) {
                final LogEntry entry = entries.get(i);
                // Set checksum after checkAndResolveConflict
                if (this.raftOptions.isEnableLogEntryChecksum()) {
                    entry.setChecksum(entry.checksum());
                }
                if (entry.getType() == EntryType.ENTRY_TYPE_CONFIGURATION) {
                    Configuration oldConf = new Configuration();
                    if (entry.getOldPeers() != null) {
                        oldConf = new Configuration(entry.getOldPeers(), entry.getOldLearners());
                    }
                    final ConfigurationEntry conf = new ConfigurationEntry(entry.getId(),
                        new Configuration(entry.getPeers(), entry.getLearners()), oldConf);
                    this.configManager.add(conf);
                }
            }
            if (!entries.isEmpty()) {
                done.setFirstLogIndex(entries.get(0).getId().getIndex());
                this.logsInMemory.addAll(entries);
            }
            done.setEntries(entries);

            doUnlock = false;
            if (!wakeupAllWaiter(this.writeLock)) {
                notifyLastLogIndexListeners();
            }

            // publish event out of lock
            this.diskQueue.publishEvent((event, sequence) -> {
              event.reset();
              event.type = EventType.OTHER;
              event.done = done;
            });
        } finally {
            if (doUnlock) {
                this.writeLock.unlock();
            }
        }
    }
}
```

- LogStorage刷磁盘
  默认批量写入磁盘文件RocksDB。
  `class RocksDBLogStorage implements LogStorage`
```java
private class AppendBatcher {
    List<StableClosure> storage;
    int                 cap;
    int                 size;
    int                 bufferSize;
    List<LogEntry>      toAppend;
    LogId               lastId;

    public AppendBatcher(final List<StableClosure> storage, final int cap, final List<LogEntry> toAppend,
                         final LogId lastId) {
        super();
        this.storage = storage;
        this.cap = cap;
        this.toAppend = toAppend;
        this.lastId = lastId;
    }

    LogId flush() {
        if (this.size > 0) {
            this.lastId = appendToStorage(this.toAppend);
            for (int i = 0; i < this.size; i++) {
                this.storage.get(i).getEntries().clear();
                Status st = null;
                try {
                    if (LogManagerImpl.this.hasError) {
                        st = new Status(RaftError.EIO, "Corrupted LogStorage");
                    } else {
                        st = Status.OK();
                    }
                    this.storage.get(i).run(st);
                } catch (Throwable t) {
                    LOG.error("Fail to run closure with status: {}.", st, t);
                }
            }
            this.toAppend.clear();
            this.storage.clear();

        }
        this.size = 0;
        this.bufferSize = 0;
        return this.lastId;
    }

    void append(final StableClosure done) {
        if (this.size == this.cap || this.bufferSize >= LogManagerImpl.this.raftOptions.getMaxAppendBufferSize()) {
            flush();
        }
        this.storage.add(done);
        this.size++;
        this.toAppend.addAll(done.getEntries());
        for (final LogEntry entry : done.getEntries()) {
            this.bufferSize += entry.getData() != null ? entry.getData().remaining() : 0;
        }
    }
}
```

### 常见配置方案

1. 本地配置
  配置统一写在配置文件里，如application.properties，通过重新打包和发布修改配置。
  这种方式已经很少使用。

2. 配置中心 + 静态发布
  这种方式还是使用配置文件。只是在CD发布的过程中，将配置文件中的占位符(比如${app.token})替换成配置中心的具体值。通过重新发布修改配置，但是不用重新打包。
  配合K8s容器的使用，这种方式甚至比动态下发更有优势，比如更加可靠和稳定。

3. 配置中心 + 动态下发
  服务的配置文件application.properties中，只有一个业务相关的配置即配置中心的地址。其他业务配置全部从配置中心获取。
  服务启动时通过getConfig查询当前值，服务运行过程中通过addListener监听配置变更值。
