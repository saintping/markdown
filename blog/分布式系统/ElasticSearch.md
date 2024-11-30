### 简介
ELK套件elasticsearch + logstash + kibana在全文搜索，日志分析，性能监控方面广泛使用。[https://github.com/elastic](https://github.com/elastic "https://github.com/elastic")

### ES架构
![es.png](https://ping666.com/wp-content/uploads/2024/10/es.png "es.png")

Lucene是一个Jar包，提供分词、倒排索引、检索等基本功能。核心功能参见样例[https://lucene.apache.org/core/9_12_0/core/index.html](https://lucene.apache.org/core/9_12_0/core/index.html "https://lucene.apache.org/core/9_12_0/core/index.html")
开源项目地址[https://github.com/apache/lucene](https://github.com/apache/lucene "https://github.com/apache/lucene")

ElasticSearch将Lucene封装成分布式的服务，通过Restful API对外提供服务（默认端口9200）。ES集群不依赖Zookeeper等外部组件，使用自身模块管理，默认端口9300。

### 基本概念
ElasticSearch作为全文搜索引擎，和数据库有很多类似之处。

- 索引
  一个索引对应数据库中的一张表。
- 文档
  一个文档对应表中的一行，一条记录。
- 字段
  一个字段对应表中的一列。
- 分片
  数据需要几个备份。创建索引时确认，无法修改。

### 部署

- 使用单独用户
```bash
groupadd es
useradd -g es es
passwd es
```

- 部署elasticsearch
  解压后运行即可。
```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.15.2-linux-x86_64.tar.gz
tar -zxf elasticsearch-8.15.2-linux-x86_64.tar.gz
cd elasticsearch-8.15.2-linux-x86_64/bin
./elasticsearch -d
```

- 添加用户角色
  默认开启了用户名和tsl验证。这里先通过配置文件关闭tsl功能。
```bash
./elasticsearch-users useradd es
./elasticsearch-users roles -a superuser es
./elasticsearch-users roles -a kibana_system es
```

- 验证
```bash
curl --cacert /data/es/elasticsearch-8.15.2/config/certs/http_ca.crt -u es http://localhost:9200
Enter host password for user 'es':
{
  "name" : "node-1",
  "cluster_name" : "my-application",
  "cluster_uuid" : "b-SvAFtZR_O3NQhDEvpthg",
  "version" : {
    "number" : "8.15.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "98adf7bf6bb69b66ab95b761c9e5aadb0bb059a3",
    "build_date" : "2024-09-19T10:06:03.564235954Z",
    "build_snapshot" : false,
    "lucene_version" : "9.11.1",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

- 部署kibana
  图形界面，解压后运行即可。使用内置的nodejs服务，默认端口5601。
```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.15.2-linux-x86_64.tar.gz
tar -zxf kibana-8.15.2-linux-x86_64.tar.gz
cd kibana-8.15.2-linux-x86_64/bin
nohup ./kibana &
```

### kibana使用
![es-dev.png](https://ping666.com/wp-content/uploads/2024/10/es-dev.png "es-dev.png")