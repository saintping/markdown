### 简介
Hadoop三大模块，HDFS、MapReduce、Yarn。HDFS是Hadoop Distributed File System的缩写，一个分布式的文件系统。
Hadoop官方文档[https://hadoop.apache.org/docs/stable/](https://hadoop.apache.org/docs/stable/ "https://hadoop.apache.org/docs/stable/")

### 单机文件系统
在看HDFS之前，我回顾了一下Linux单机文件系统ext4的设计，以及Group Descriptors、inode Table和Data Blocks等概念。

![ext4-layout.png](https://ping666.com/wp-content/uploads/2024/09/ext4-layout.png "ext4-layout.png")

原文在这里[https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html](https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html "https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html")

下面Ext4文件系统的几个重要设计目标，以及在分布式环境下的作用分析

- Compatibility
  兼容之前的版本Ext2和Ext3，这个是Ext4特有的
- Bigger File System and File Sizes
  大，更大，分布式文件系统天生支持scale up
- Sub directory scalability
  同上
- Fast fsck
  Namenode上的FSImage也需要
- Journal checksumming
  有一点用，分布式文件系统也可以用Ext4做载体
- Online defragmentation
  变的不那么重要，大数据场景都是LSM型记录，碎片化不那么严重

从上面的对比可以看出来，HDFS在文件系统这一层的复杂度比单机文件系统要低。但是要处理额外的跨机交互，负载均衡，HA热备等。

### HDFS架构
下图是HDFS的架构图。
![hdfs-architecture.png](https://ping666.com/wp-content/uploads/2024/09/hdfs-architecture.png "hdfs-architecture.png")

NameNode节点上对应的是Group Descriptors、inode Table，Datanodes节点上对应的是Data Blocks。
上图没有把文件分块和备份表示出来，见下图。
![hdfs-datanodes.png](https://ping666.com/wp-content/uploads/2024/09/hdfs-datanodes.png "hdfs-datanodes.png")

### NameNode和DataNode
从上面架构图了可以看到，每个文件块在DataNode上是有多个备份的（可配置），所以DataNode机器可以随时故障下线，文件块的备份状态完全由NameNode维护。NameNode是整个集群的单点，通过HA热备来保证可靠性。

NameNode核心结构FSDirectory保存了文件系统的所有元数据，FSDirectory完全在内存以提高集群性能，同时备份到磁盘文件FSImage上。
主NameNode负责写内存数据FSDirectory和记录Edit日志，备份的SecondNameNode负责定期将Edit日志合并到FSImage，也就是Checkpoint。SecondNameNode执行完Checkpoint后通知NameNode切换到新版本的FSImage。集群通过Checkpoint+Edit日志实现快速的故障恢复。
这部分代码在
`hadoop\hadoop-hdfs-project\hadoop-hdfs\src\main\java\org\apache\hadoop\hdfs\server\namenode\SecondaryNameNode.java`
```java
  public boolean doCheckpoint() throws IOException {
    checkpointImage.ensureCurrentDirExists();
    NNStorage dstStorage = checkpointImage.getStorage();
    
    // Tell the namenode to start logging transactions in a new edit file
    // Returns a token that would be used to upload the merged image.
    CheckpointSignature sig = namenode.rollEditLog();
    
    boolean loadImage = false;
    boolean isFreshCheckpointer = (checkpointImage.getNamespaceID() == 0);
    boolean isSameCluster =
        (dstStorage.versionSupportsFederation(NameNodeLayoutVersion.FEATURES)
            && sig.isSameCluster(checkpointImage)) ||
        (!dstStorage.versionSupportsFederation(NameNodeLayoutVersion.FEATURES)
            && sig.namespaceIdMatches(checkpointImage));
    if (isFreshCheckpointer ||
        (isSameCluster &&
         !sig.storageVersionMatches(checkpointImage.getStorage()))) {
      // if we're a fresh 2NN, or if we're on the same cluster and our storage
      // needs an upgrade, just take the storage info from the server.
      dstStorage.setStorageInfo(sig);
      dstStorage.setClusterID(sig.getClusterID());
      dstStorage.setBlockPoolID(sig.getBlockpoolID());
      loadImage = true;
    }
    sig.validateStorageInfo(checkpointImage);

    // error simulation code for junit test
    CheckpointFaultInjector.getInstance().afterSecondaryCallsRollEditLog();

    RemoteEditLogManifest manifest =
      namenode.getEditLogManifest(sig.mostRecentCheckpointTxId + 1);

    // Fetch fsimage and edits. Reload the image if previous merge failed.
    loadImage |= downloadCheckpointFiles(
        fsName, checkpointImage, sig, manifest) |
        checkpointImage.hasMergeError();
    try {
      doMerge(sig, manifest, loadImage, checkpointImage, namesystem);
    } catch (IOException ioe) {
      // A merge error occurred. The in-memory file system state may be
      // inconsistent, so the image and edits need to be reloaded.
      checkpointImage.setMergeError();
      throw ioe;
    }
    // Clear any error since merge was successful.
    checkpointImage.clearMergeError();

    
    //
    // Upload the new image into the NameNode. Then tell the Namenode
    // to make this new uploaded image as the most current image.
    //
    long txid = checkpointImage.getLastAppliedTxId();
    TransferFsImage.uploadImageFromStorage(fsName, conf, dstStorage,
        NameNodeFile.IMAGE, txid);

    // error simulation code for junit test
    CheckpointFaultInjector.getInstance().afterSecondaryUploadsNewImage();

    LOG.warn("Checkpoint done. New Image Size: " 
             + dstStorage.getFsImageName(txid).length());

    if (legacyOivImageDir != null && !legacyOivImageDir.isEmpty()) {
      try {
        checkpointImage.saveLegacyOIVImage(namesystem, legacyOivImageDir,
            new Canceler());
      } catch (IOException e) {
        LOG.warn("Failed to write legacy OIV image: ", e);
      }
    }
    return loadImage;
  }
```

在HA的备机SecondNameNode上，会持续将Edit日志合并到FSImage，同时定期做Checkpoint。SecondNameNode维护的Checkpoint+Edit日志保证了快速的错误恢复。

Edit日志操作FSEditLogOp的继承关系如下图：
![FSEditLogOp.jpg](https://ping666.com/wp-content/uploads/2024/09/FSEditLogOp.jpg "FSEditLogOp.jpg")

### HDFS命令执行过程
`hadoop fs -put localfile /user/hadoop/hadoopfile`
上面这条命令，背后的执行过程是怎样的呢？
hadoop只是一个bash脚本，-put对应的类是org.apache.hadoop.fs.shell.CopyCommands.Put，最终被翻译成如下对DFSClient的调用：

1. 在NameNode上创建文件
1. 以块为单位向NameNode申请写入流，NameNode协调DataNode上的块存储位置（一般是找靠近client的）
1. 客户端从本地文件复制数据到DataNode的目标块
1. 块数据写成功后，DataNode向NameNode发送ACK
1. 每个块写入都重复上面3步，直到整个文件数据全部写完
1. 客户端向NameNode发起请求确认该文件状态
1. NameNode确认DataNode所有块写入完毕，文件状态正常
1. 客户端命令执行完毕退出

HDFS常用命令参见[https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/FileSystemShell.html](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/FileSystemShell.html "https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/FileSystemShell.html")

### CEPH
从上面可以看到HDFS的NameNode是有单点的，但在Hadoop大文件、批处理场景下，这个单点问题不是特别严重。所以在面向C端用户的高并发大量小文件场景，一般都使用类似CEPH的分布式文件系统。
[https://docs.ceph.com/en/latest](https://docs.ceph.com/en/latest "https://docs.ceph.com/en/latest")

CEPH核心数据结构如下：
![https://ping666.com/wp-content/uploads/2024/12/ceph-crush.png](https://ping666.com/wp-content/uploads/2024/12/ceph-crush.png "ceph-crush.png")

- File和HDFS里的File概念类似，都是指面向用户的文件
- Object类似HDFS里的Block
  CEPH和HDFS都会将用户文件映射成块列表。
- OSD（Object Storage Device）负责数据块的物理存储，类似HDFS里的DataNode
- PG（Placement Group）是一个逻辑概念。每个PG和OSD有明确的对应关系，如果是3个备份则每个PG对应3个OSD。
- CRUSH MAP，CEPH集群的拓扑结构（包括机架、主机、OSD及其容量）

##### CEPH寻址

CEPH寻址过程和传统的集中式Metadata映射（比如HDFS的NameNode）的算法不同。
这部分代码在`src\crush\mapper.c`

1. 从Object映射到PG
   这一步就是简单的Hash过程。hash（object_key）& pg_size= pgid。
   因为这一步对PG数量非常敏感，所以对PG扩容是不太可能的。好在PG只是一个逻辑概念，一般部署时就已经确定了。
1. 从PG映射到OSD
   这一步就是CEPH特有的算法，CRUSH（Controlled Replication Under Scalable Hashing）
   CRUSH可以看成是一个特殊的映射算法，他需要满足以下条件：
   1. 输入为PGID，CRUSH MAP。
   1. 输出多个OSD ID，一般为3个备份。
   1. 结果OSD需要满足一些条件，比如尽量离散，不能在同一个主机，不能在同一个机架等。

CRUSH算法是CEPH做到真正分布式的关键。算法细节参考[http://www.ssrc.ucsc.edu/papers/weil-sc06.pdf](http://www.ssrc.ucsc.edu/papers/weil-sc06.pdf "http://www.ssrc.ucsc.edu/papers/weil-sc06.pdf")

因为在同样的CRUSH MAP下，Location映射的输出只和Object Key相关。所以CEPH客户端可以直接向目标OSD发起文件内容的读写，而不需要经过一个统一的代理或者集群入口。

Redis Cluster模式也是类似的一致性Hash思路，只是Redis集群的拓扑结构更加简单，只有槽号和主机的映射关系。Redis通过Gossip协议在集群内的所有节点广播拓扑结构，Redis的所有节点都可以直接响应客户端请求。CEPH则通过任意数量的Monitor程序来维护拓扑结构，客户端通过Monitor拿到OSD后，直接向OSD发起文件内容的读写。

##### CEPH扩容和缩容
一致性Hash算法，复杂的地方在集群扩容和缩容。CEPH的扩容和缩容，一般表现为OSD的增加和删除，CRUSH MAP拓扑结构的变化。
```bash
ceph orch daemon add osd host1:/dev/sdb
ceph orch osd rm stop 4
```
新旧CRUSH MAP的变化，会导致部分PG对OSD映射的变化，这个过程会涉及到数据的迁移。