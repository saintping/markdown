### 前言
Mysql数据库InnoDB引擎B+树的性质，决定了树的高度是表性能的关键因数。一般4层已经是极限了（每一层多一次磁盘查询，耗时会明显上升），大概亿级别的记录数。当业务量大于这个量级，就需要考虑分库分表方案了。

分库分表分为垂直和水平两种思路，垂直分库分表更多的涉及业务的数据规划，和技术没有太大的关系。一般讲的都是水平方式。如果只是把Mysql当KV存储，可以简单的用主键Hash找到数据源和表。在应用层做一个切面配合注解，在SQL执行前替换掉DataSource和表名就可以了。不过这样会有两个明显的问题：

- 代码侵入
  分库分表的逻辑会散落在应用服务任何地方。并且外围工具也需要支持分库分表逻辑。
- 跨表操作
  肯定还是很多场景，需要跨表的。比如跨表查询和事务处理。这个在业务层做就非常麻烦了。

ShardingSphere是一个开源的数据库代理方案。他重写了JDBC层，实现真正的非侵入式增强。支持分库分表、读写分离、数据加密脱敏、影子数据库等。
[https://github.com/apache/shardingsphere](https://github.com/apache/shardingsphere "https://github.com/apache/shardingsphere")

### ShardingSphere架构

![shardingsphere-proxy.png](https://ping666.com/wp-content/uploads/2024/11/shardingsphere-proxy.png "shardingsphere-proxy.png")

在应用层完全可以把数据库代理服务shardingsphere-proxy看成是一个数据库，只需要改一下JDBC串就可以了。代码不用做任何改动就可以支持分库分表。

ShardingSphere还提供了另外一种嵌入式的方案shardingsphere-jdbc，将shardingsphere-jdbc作为Jar包集成到应用服务里。

Product环境推荐使用Proxy方案，有以下优势：

- 逻辑解耦
  虽然请求增加了一跳，延迟会略微高一点。但是好处是分库分表逻辑完全解耦了（特别是事务那一块，因为分库分表后一般都涉及到分布式事务的开启）。应用层/工具集都不需要修改任何代码。
- 优化连接池
  数据库的连接是一个重资源。业务量以及需要分库分表了，这时TPS肯定也低不了。假设前面有100台机器，每台机器连接数是100个，这样一个数据库实例可能就挂了10000个连接，这里面会有很多空闲的连接。造成了极大的浪费。Proxy模式可以让每个连接都不空闲。
- 数据治理
  Proxy上提供了丰富的数据治理功能，这是作为插件的shardingsphere-jdbc无法提供的。

### 代理接入
这里以shardingsphere-jdbc为例，因为Porxy模式只是简单启动一个服务，太简单了。

1. 加入Jar包依赖
```xml
<dependency>
	<groupId>org.apache.shardingsphere</groupId>
	<artifactId>shardingsphere-jdbc</artifactId>
	<version>5.5.1</version>
</dependency>
```
1. 将DataSource切换到ShardingSphereDriver
```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:sharding-jdbc.yaml
```
1. shardingsphere-jdbc配置文件
  所有的分库分表逻辑统一在这个配置文件里。
![sharding-jdbc.yaml_.png](https://ping666.com/wp-content/uploads/2024/11/sharding-jdbc.yaml_.png "sharding-jdbc.yaml_.png")

然后业务代码就能自动做到分库分表了。无论是用Hibernate/MyBatis，还是JDBCTemplate写的。

**注意：**

1. 高版本已经废弃了shardingsphere-jdbc-core-spring-boot-starter
  可以参见这里[https://github.com/apache/shardingsphere/issues/22469](https://github.com/apache/shardingsphere/issues/22469 "https://github.com/apache/shardingsphere/issues/22469")
  一句话，维护不过来。
1. 不要格式化sharding-jdbc.yaml文件！！！

### SQL案例
ShardingSphere在JDBC层，将能定向数据源的SQL直接转发SQL过去，将需要跨表操作的分别发SQL过去，然后在内存中聚合结果。不过他不是支持所有JDBC接口，比如不支持存储过程等。

- insert
```bash
ShardingSphere-SQL: Logic SQL: insert into t_student (age,name) values (?,?)
ShardingSphere-SQL: Actual SQL: test_2 ::: insert into t_student_1 (age,name, id) values (?, ?, ?) ::: [10, abc, 1068265101313703936]
ShardingSphere-SQL: Logic SQL: insert into t_student (age,name) values (?,?)
ShardingSphere-SQL: Actual SQL: test_1 ::: insert into t_student_2 (age,name, id) values (?, ?, ?) ::: [10, abc, 1068265565606379521]
```

- select
```bash
#主键查询
ShardingSphere-SQL: Logic SQL: select se1_0.id,se1_0.age,se1_0.name from t_student se1_0 where se1_0.id=?
ShardingSphere-SQL: Actual SQL: test_2 ::: select se1_0.id,se1_0.age,se1_0.name from t_student_1 se1_0 where se1_0.id=? ::: [1068265101313703936]
#非主键查询
ShardingSphere-SQL: Actual SQL: test_1 ::: select se1_0.id,se1_0.age,se1_0.name from t_student_1 se1_0 where se1_0.age=? UNION ALL select se1_0.id,se1_0.age,se1_0.name from t_student_2 se1_0 where se1_0.age=? ::: [100, 100]
ShardingSphere-SQL: Actual SQL: test_2 ::: select se1_0.id,se1_0.age,se1_0.name from t_student_1 se1_0 where se1_0.age=? UNION ALL select se1_0.id,se1_0.age,se1_0.name from t_student_2 se1_0 where se1_0.age=? ::: [100, 100]
#带聚合的查询
ShardingSphere-SQL: Logic SQL: select avg(se1_0.age) from t_student se1_0
ShardingSphere-SQL: Actual SQL: test_1 ::: select avg(se1_0.age) , COUNT(se1_0.age) AS AVG_DERIVED_COUNT_0 , SUM(se1_0.age) AS AVG_DERIVED_SUM_0 from t_student_1 se1_0 UNION ALL select avg(se1_0.age) , COUNT(se1_0.age) AS AVG_DERIVED_COUNT_0 , SUM(se1_0.age) AS AVG_DERIVED_SUM_0 from t_student_2 se1_0
ShardingSphere-SQL: Actual SQL: test_2 ::: select avg(se1_0.age) , COUNT(se1_0.age) AS AVG_DERIVED_COUNT_0 , SUM(se1_0.age) AS AVG_DERIVED_SUM_0 from t_student_1 se1_0 UNION ALL select avg(se1_0.age) , COUNT(se1_0.age) AS AVG_DERIVED_COUNT_0 , SUM(se1_0.age) AS AVG_DERIVED_SUM_0 from t_student_2 se1_0
```

- update
```bash
# 主键更新
ShardingSphere-SQL: Logic SQL: update t_student set age=?,name=? where id=?
ShardingSphere-SQL: Actual SQL: test_2 ::: update t_student_1 set age=?,name=? where id=? ::: [10, efg, 1068265101313703936]
# 非主键更新
ShardingSphere-SQL: Logic SQL: update t_student se1_0 set age=?
ShardingSphere-SQL: Actual SQL: test_1 ::: update t_student_1 se1_0 set age=? ::: [100]
ShardingSphere-SQL: Actual SQL: test_1 ::: update t_student_2 se1_0 set age=? ::: [100]
ShardingSphere-SQL: Actual SQL: test_2 ::: update t_student_1 se1_0 set age=? ::: [100]
ShardingSphere-SQL: Actual SQL: test_2 ::: update t_student_2 se1_0 set age=? ::: [100]
```

- 正常的单表操作不受影响
```bash
ShardingSphere-SQL: Logic SQL: select se1_0.id,se1_0.name from t_school se1_0 where se1_0.id=?
ShardingSphere-SQL: Actual SQL: test ::: select se1_0.id,se1_0.name from t_school se1_0 where se1_0.id=? ::: [1068265101313703936]
```

### 代码实现

应用程序在调用JDBC的接口执行SQL（比如PreparedStatement::executeQuery）时，会被ShardingSpherePreparedStatement::executeQuery接管。

- 路由算法
  ShardingSpherePreparedStatement::executeQuery这里先会调用到ShardingSphere核心处理程序KernelProcessor.route，通过配置的路由策略doSharding映射目标数据库和表。
```java
public final class InlineShardingAlgorithm implements StandardShardingAlgorithm<Comparable<?>> {
    @Override
    public String doSharding(final Collection<String> availableTargetNames, final PreciseShardingValue<Comparable<?>> shardingValue) {
        ShardingSpherePreconditions.checkNotNull(shardingValue.getValue(), NullShardingValueException::new);
        String columnName = shardingValue.getColumnName();
        ShardingSpherePreconditions.checkState(algorithmExpression.contains(columnName), () -> new MismatchedInlineShardingAlgorithmExpressionAndColumnException(algorithmExpression, columnName));
        try {
            return InlineExpressionParserFactory.newInstance(algorithmExpression).evaluateWithArgs(Collections.singletonMap(columnName, shardingValue.getValue()));
        } catch (final MissingMethodException ignored) {
            throw new MismatchedInlineShardingAlgorithmExpressionAndColumnException(algorithmExpression, columnName);
        }
    }
}
```
InlineExpressionParserFactory会调用SPI接口org.apache.shardingsphere.infra.expr.groovy.GroovyInlineExpressionParser去计算路由规则。

- SQL执行
  将SQL并行提交到数据库实例上去执行，然后做结果聚合。ShardingDALResultMerger/ShardingDDLResultMerger/ShardingDQLResultMerger/TransparentResultMerger。这里不同的SQL聚合逻辑有很大的不同（特别是涉及到group分组的计算）。

### 分布式事务

![shardingsphere-transaction.webp](https://ping666.com/wp-content/uploads/2024/11/shardingsphere-transaction.webp "shardingsphere-transaction.webp")

ShardingSphere支持三种类型的分布式事务：

- TransactionType.LOCAL（默认方式）
  使用弱XA的1PC提交，没有prepare阶段（减少了资源锁定时间）。部分情况下无法做到回滚。

- TransactionType.XA
  使用XA的2PC提交，强一致性（CP）事务协议，默认使用atomikos实现。所有关系型数据库都支持XA协议，部分消息队列如RocketMQ也支持。
```sql
XA BEGIN tid;
XA END tid;
XA PREPARE tid;
XA COMMIT tid;
XA ROLLBACK tid;
XA RECOVER FORMAT='SQL';
```

- TransactionType.BASE
  BASE就是AP型的事务，为了性能牺牲了部分一致性，核心是回滚时的补偿逻辑。使用阿里Seata实现，有三种补偿模式AT、TTC、SAGA。

XA对事务性能的影响是非常明显的。

![shardingsphere-transaction-performance.webp](https://ping666.com/wp-content/uploads/2024/11/shardingsphere-transaction-performance.webp "shardingsphere-transaction-performance.webp")

如果开启分布式事务支持比如XA和BASE，建议使用Proxy模式，以和应用事务的分开。

### Proxy与数据治理

- DistSQL
  核心能力是支持在线修改各种配置，不用重启服务。
```sql
SHOW STORAGE UNITS; //对应配置文件里的dataSources，还有创建删除更新等。
SHOW SHARDING TABLE RULES; //对应配置文件里的rules.tables
SHOW SHARDING ALGORITHMS; //对应配置文件里的rules.shardingAlgorithms
SHOW TRANSACTION RULE; //默认XA
SHOW COMPUTE NODE INFO; //查看当前Proxy实例列表
EXPORT DATABASE CONFIGURATION； //将当前配置以yaml格式导出
IMPORT DATABASE CONFIGURATION； //导入yaml格式配置
```

- 数据迁移
  这里的方案，新老数据库切换时，会涉及到短暂的数据不可写。迁移文档在这里。[https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/migration/usage](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/migration/usage "https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/migration/usage")
```sql
MIGRATE TABLE ds.t_order INTO sharding_db.t_order; //ds.t_order是分库分表前的数据源，sharding_db.t_order是分库分表后的逻辑库表
SHOW MIGRATION LIST; //查看当前迁移任务
SHOW MIGRATION STATUS 'j0102p00002333dcb3d9db141cef14bed6fbf1ab54'; //迁移状态
SHOW MIGRATION CHECK STATUS 'j0102p00002333dcb3d9db141cef14bed6fbf1ab54'; //检查数据一致性
COMMIT MIGRATION 'j0102p00002333dcb3d9db141cef14bed6fbf1ab54'; //停止迁移
```
