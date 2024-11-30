### 前言
Hive在传统数据仓库场景工作的很好，为什么后面被Spark抢尽风头呢？主要是因为机器学习太火了。

一般的机器学习，就是暴力解方程Y = WX + b。其中X是大量的样本集合，Y是结果集合，b偏差，W是需要求解的参数集。
一般的机器学习（比如分类和回归）过程如下：
1. 始化参数W=W1
2. 计算W1X + b得到预测值Y1
3. 通过Lost函数计算损失值
4. 根据损失值计算损失函数的梯度，更新模型X以减少损失值
5. 一直迭代重复上面的2、3、4步，直到符合停止条件(比如迭代次数到达上限或者损失值无法继续降低)

这里的迭代次数动辄都是百万次的，Hive每计算一步都将结果落回磁盘HDFS文件，下一步又从磁盘加载回来的做法，在这个场景下是致命的。

Spark通过定义RDD（Resilient Distributed Datasets，弹性分布式数据集），将计算结果一直放到内存里，性能直接秒了Hive。随着Spark的不断发展，后续又定义了更高层级的抽象DataFrame和Dataset，将这些抽象映射到Python和R接口后，带着他们在机器学习领域起飞。

### Spark架构
![spark.jpg](https://ping666.com/wp-content/uploads/2024/09/spark.jpg "spark.jpg")

RDD支持的操作很像Hive的*Operator，比如filter、distinct、sortBy、union，也有特别的如persist、cache。

详情参见[https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html "https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html")

### Spark API
Spark主要是以Scala语言写的，就不看具体的源码了。

Spark的主流用法还是从[Python](https://archive.apache.org/dist/spark/docs/3.0.0/api/python/index.html "Python")、Scala、R、Java等API开始的，因为机器学习算法都是以基础库的方式开源提供。

Spark也提供SQL入口，但是Spark SQL和Hive SQL不是一个东西。所以传统的大数据仓库，一般是以Hive over Spark的方式来使用。这样业务还是只写Hive SQL，至于运行是选择Hive引擎还是Hive over Spark，就看具体的场景考虑了。
