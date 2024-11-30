### 前言
前段时间做Azkaban跑批任务循环依赖检测的时候，使用的是jgrapht。将所有跑批任务的依赖关系导入jgrapht组成一个全局的DAG图，然后使用jgrapht的dijkstra等算法来查找最短路径和循环路径等。

jgrapht内置的图形显示模块很弱，需要通过DOT格式外接Graphviz来显示图形。尝试使用专业的图数据库Neo4j来做这个工作。
>Neo4j is the world’s leading Graph Database. It is a high performance graph store with all the features expected of a mature and robust database, like a friendly query language and ACID transactions.

开源项目地址[https://github.com/neo4j/neo4j.git](https://github.com/neo4j/neo4j.git "https://github.com/neo4j/neo4j.git")

### Neo4j架构

![neo4j-architecture.png](https://ping666.com/wp-content/uploads/2024/11/neo4j-architecture.png "neo4j-architecture.png")

Neo4j的架构比较简单，既可以作为嵌入式数据库使用，也可以作为独立的服务启动。
图数据库和传统关系型数据最大的不同是没有表的概念。一个库就是一个图。

- 节点Nodes
  图形中的顶点
- 关系Relationships
  图形中的边，有方向。
- 节点标签Labels
  可以有多个，查询的时候方便检索
- 节点属性Property Keys
  可以有多个，查询的时候方便检索
- 关系类型Relationship Types
  可以有多个，查询的时候方便检索

```sql
(:nodes)-[:ARE_CONNECTED_TO]->(:otherNodes)
```
以上三元组是图数据库的核心数据结构，小括号表示节点，中括号表示关系，->表示关系的方向。可选的大括号如{name:'Tom Cruise'}表示节点的属性。

### 核心存储结构
有些关系型数据库使用三元组的方案存储图数据，即每条关系是表中的一行记录。这样多层关系查询的时候，每一层都需要一个Join操作，数据量稍微大一点，性能是是无法接受的。

原生的图数据库使用一种免索引邻接（index-free adjacency) 的结构。三元组还是核心的数据结构，但是不同关系上的相同节点都组成了双向链表。这是图原生数据库性能高的关键。

![neo4j-data-struct.png](https://ping666.com/wp-content/uploads/2024/11/neo4j-data-struct.png "neo4j-data-struct.png")

存储结构的代码相关的代码在org.neo4j.kernel.impl.store.record，NodeRecord、PropertyRecord、RelationshipRecord等。不同的Record保存在不同的文件里，所以节点文件和关系文件的记录是定长的，Record的id和文件偏移量有对应关系。

```java
public class RelationshipRecord extends PrimitiveRecord { //父类上有一个long id; 代表这个记录的key，因为节点和关系是定长结构，所以id对应记录在文件上的偏移量
    public static final long SHALLOW_SIZE = shallowSizeOfInstance(RelationshipRecord.class);
    private long firstNode; //节点1
    private long secondNode; //节点2
    private int type; //关系类型
    private long firstPrevRel; //节点1的前一个关系
    private long firstNextRel;  //节点1的后一个关系
    private long secondPrevRel;  //节点2的前一个关系
    private long secondNextRel;  //节点2的后一个关系
    private boolean firstInFirstChain;
    private boolean firstInSecondChain;
}
```

### 安装与使用
```bash
yum install java-17-konajdk-devel.x86_64
rpm --import https://debian.neo4j.com/neotechnology.gpg.key
cat /etc/yum.repos.d/neo4j.repo
[neo4j]
name=Neo4j
baseurl=http://yum.neo4j.com/stable
enabled=1
gpgcheck=1
yum install neo4j.noarch cypher-shell.noarch
```
>You are using an unsupported version of the Java runtime. Please use Oracle(R) Java(TM) 11 or OpenJDK(TM) 11.

指定需要这两种虚拟机，腾讯云的TencentKonaJDK不符合。使用Windows的安装包[https://neo4j.com/artifact.php?name=neo4j-desktop-offline-1.6.1-setup.exe](https://neo4j.com/artifact.php?name=neo4j-desktop-offline-1.6.1-setup.exe "https://neo4j.com/artifact.php?name=neo4j-desktop-offline-1.6.1-setup.exe")

安装后自带了一个样例数据库演示各种功能，比如类SQL查询语言Cypher。

### 查询语言Cypher

>Cypher is Neo4j’s declarative and GQL conformant query language. Available as open source via The openCypher project, Cypher is similar to SQL, but optimized for graphs.

- CREATE建立节点和关系
  可以先建立节点，再建立关系，也可以直接一步建立节点和关系。
![neo4j-cypher-create.png](https://ping666.com/wp-content/uploads/2024/11/neo4j-cypher-create.png "neo4j-cypher-create.png")

- Match查询
![neo4j-cypher-query.png](https://ping666.com/wp-content/uploads/2024/11/neo4j-cypher-query.png "neo4j-cypher-query.png")

Neo4j是Java写的，有两套API。一般使用Cypher接口操作数据库。
`implementation 'org.neo4j:neo4j:5.24.0'`

### 支持向量

Neo4j在5.13版本增加了对向量的支持。向量数据库是AI领域LLM Agent等的核心组件。支持向量存储的两个关键点：

- 向量非常大
  普通的一行文本、一段语音、一张图，特征值的维度几千、上万已经是常态。
```sql
CREATE VECTOR INDEX moviePlots IF NOT EXISTS
FOR (m:Movie)
ON m.embedding
OPTIONS { indexConfig: {
 `vector.dimensions`: 1536,
 `vector.similarity_function`: 'cosine'
}}
```

- 内置向量相似性算法
  近邻算法（K-Nearest Neighbors），欧几里得相似和余弦相似度（Cosine Similarity）等。默认使用cosine算法。
```sql
MATCH (m:Movie {title: 'Godfather, The'})
CALL db.index.vector.queryNodes('moviePlots', 5, m.embedding)
YIELD node AS movie, score
RETURN movie.title AS title, movie.plot AS plot, score
```


