### 基本语法
Hive的详细语法见官方文档。
[https://cwiki.apache.org/confluence/display/Hive/LanguageManual](https://cwiki.apache.org/confluence/display/Hive/LanguageManual "https://cwiki.apache.org/confluence/display/Hive/LanguageManual")

### 数据类型

- 整数类型：TINYINT、SMALLINT、INT、BIGINT
- 浮点类型：FLOAT、DOUBLE、DECIMAL
- 文本类型：STRING、CHAR、VARCHAR
- 时间类型：DATE、TIMESTAMP、INTERVAL
- 布尔类型：BOOLEAN
- 二进制类型：BINARY
- 复杂类型：ARRAY、MAP、STRUCT、UNIONTYPE

一般很少直接将表字段定义成复杂类型。但是复杂类型在行转列、列转行的场景下，作为中间类型使用非常方便。

有两个特别需要注意的地方：

1. 类型隐式转换
   Hive计算过程中，不同的类型会做隐式转换，不过有很多坑，建议使用cast显示转换。
2. null比较也要特别注意
   null值和其他任何类型做比较都是未定义的，甚至两个null比较也是这样。需要用is null和nvl先对null值做特殊处理。比如下面的sql，如果col是null值，那么无论是任何类型，返回的都是4
```sql
select
    case
        when col > '1' then 1
        when col = '1' then 2
        when col < '1' then 3
        else 4
    end as result
```

类型隐式转换全图如下：
![hive-type-conversion.jpg](https://ping666.com/wp-content/uploads/2024/09/hive-type-conversion.jpg "hive-type-conversion.jpg")

### 聚合和开窗
Hive提供了100多个内置函数，以方便使用。重点说一下聚合函数。
```sql
SELECT
    class,
    COUNT(1) AS num,
    AVG(score) AS score_avg
FROM student
    GROUP BY class;
```
上面这个SQL算每个班的学生人数和平均分。但是一般业务场景不会这么简单，都会有各种复杂条件。比如，找到每个班的前10名，并且算一下他们的平均分。这就需要开窗了。开窗的本质是在聚合之前赋予记录一些特征，以在聚合时可以区别对待。
```sql
SELECT
    class,
    SUM(CASE WHEN rank <= 10 THEN 1 ELSE 0 END) AS num,
    AVG(CASE WHEN rank <= 10 THEN score ELSE NULL END) AS score_avg
FROM
    (
        SELECT
            class,
            core,
            ROW_NUMBER() OVER(PARTITION BY class ORDER BY score DESC) AS rank
        FROM student
    ) AS ranked_students
GROUP BY class;
```
在处理连续型分析时，LED和LAG函数非常好用。比如在上面的基础上，找到前10名中已经连续3次分数越来越高的同学。
```sql
SELECT
    DISTINCT name
FROM
(
    SELECT
        name,
        score,
        LAG(score, 1) OVER (PARTITION BY id ORDER BY score DESC) AS last_score,
        LAG(score, 2) OVER (PARTITION BY id ORDER BY score DESC) AS last2_score
    FROM
    (
        SELECT
            name,
            score,
            ROW_NUMBER() OVER (PARTITION BY class ORDER BY score DESC, id) AS rank
        FROM
            student
    ) AS ranked_students
    WHERE
        rank <= 10
) AS filtered_students
WHERE
    score > last_score
    AND (last_score IS NULL OR last_score > last2_score);
```
开窗能解决绝大部分问题，如果还不行，就多来一层。

### 自定义函数
为了适应复杂的数据分析场景，Hive支持用户自定义函数以扩展功能。自定义函数分三种

- UDF（User-Define Function）
- UDAF（User-Define Aggregate Function）
- UDTF（User-Defined Table-Generating Functions）

以下是使用自定义UDF的过程。
1. 自定义UDF
   实现UDF的evaluate方法
```java
public class AddUDF extends UDF {
    public IntWritable evaluate(Integer a, Integer b) {
        if (a == null || b == null) {
            return null;
        }
        return new IntWritable(a + b);
    }
}
```

2. 编译成jar包并且加载
   `ADD JAR /path/to/your/add-udf.jar;`
   可以在SQL里每次都临时加载，也可以加载为默认。
   
3. 在SQL中使用
```sql
create temporary function add_udf as 'com.test.hive.udf.addudf';
select add_udf(1, 2) as result;
```
