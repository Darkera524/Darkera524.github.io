---
layout: post
title:  "spark基本原理及pyspark应用"
categories: python bigdata spark
tags: python spark pyspark bigdata
author: 大可
---

## 目录
- [Spark](#spark)
    - [核心概念](#核心概念)
        - [Spark核心:弹性分布式数据集](#spark核心_弹性分布式数据集)
            - [惰性评估计算](#惰性评估计算)
            - [RDDs持久化管理](#rdds持久化管理)
            - [RDD的逻辑组成](#rdd的逻辑组成)
        - [DataFrames](#dataframes)
        - [Spark_Streaming](#spark_streaming)
        - [Job、Stage、Task](#job_stage_task)
        - [DataSet](#dataset)
    - [Catalyst优化器](#catalyst优化器)
    - [Spark_job调度](#spark_job调度)
        - [Applications间资源分配](#applications间资源分配)
        - [Spark应用分配过程](#spark应用分配过程)
    - [部署](#部署)
        - [问题汇总](#问题汇总)
- [PySpark](#pyspark)
    - [SparkContext](#sparkcontext)
        - [Master分类](#master分类)
    - [SparkSession](#sparksession)
    - [RDD](#rdd)
        - [RDD function（以下rdd代表Rdd对象）](#rdd&nbsp;function（以下rdd代表rdd对象）)
        - [RDD Persist](#rdd_persist)
    - [DataFrame](#dataframe)
        - [创建DataFrame](#创建dataframe)
        - [查询DataFrame](#查询dataframe)
        - [schema定义](#schema定义)
        - [DataFrame Persist](#dataframe_persist)
        - [数据预处理](#数据预处理)
    - [SparkStreaming](#sparkstreaming)
        - [非全局聚合（无状态流处理）](#非全局聚合（无状态流处理）)
        - [全局聚合(有状态的流处理)](#全局聚合(有状态的流处理))
        - [结构化流处理](#结构化流处理)
    - [Spark UDF](#spark-udf)
        - [原有udf方式](#原有udf方式)
        - [pandas_udf Scalar](#pandas_udf-scalar)
        - [pandas_udf grouped map](#pandas_udf-grouped-map)
        - [pandas_udf grouped aggregate](#pandas_udf-grouped-aggregate)
    - [学习过程的其他疑问Question](#学习过程的其他疑问question)
- [ref](#ref)
- [相关阅读](#相关阅读)


## Spark

### 核心概念
#### Spark核心_弹性分布式数据集
- （RDDs--Resilient Distribute Datasets）是一组不可变的JVM对象的分布集，可执行高速运算
- 该数据集是分布式的，基于某种关键字，数据集被划分成块，同时分发到执行机器节点
- RDD追踪每个块上所做的转换（记录日志），在发生错误和部分数据丢失时提供回退，重新计算数据
- 无Schema数据结构
- 组成RDD的对象称为partition，不同的partition可能会被分配到分布式系统的不同节点上去计算

##### 惰性评估计算
- 什么时候惰性
    - RDD并不会在每一个transformation去评估计算，而是采取惰性计算的方式，只有在最终数据需要被计算的时候（Action）去计算
        - transformation通过处理一个RDD产生一个新的RDD
        - Action返回的是RDD之外的对象，驱动分区的评估计算并且将结果输出到非Spark系统(这里指Spark执行节点之外)，如将数据传回Driver，或者将数据存储外额外的存储系统
    - Action会触发调度器（scheduler），调度器将基于RDD Transformations之间的依赖创建一个有向无环图，然后依据execution plan（执行计划），调度器为每一个stage计算缺少的partitions直到得到结果
    - TIP！
        - 并不是所有的Transformation都是惰性的，sortByKey触发一个Transformation和一个Action
- 惰性评估的性能考究
    - 一方面，惰性评估计算减少了Driver与Executor之间的交互，提高了效率
    - 另一方面，惰性评估计算使用了有向无环图的机制，在计算时，如果有针对同一RDD的不同操作，不必读取该RDD多次，仅在一次读取过程中完成两个操作，提高了计算效率
- 惰性评估的容错考究
    - 因为每个Transformation的结果都是RDD，再加上有向无环图的特性，RDD本身维护了本身的依赖，故当一个partition失效时，RDD拥有足够的信息来重新计算，同时该计算可以并行来使恢复过程更快
##### RDDs持久化管理
- Saprk提供了三种持久化管理的选项：内存存储反序列化数据、内存存储序列化数据、磁盘存储
    - 内存存储反序列化数据是最直接且存储最快的方式，因为省去了序列化数据的开销，但是并不是最高效的方式，要求数据被存储为objects的形式
    - 当数据在网络中传输时，Saprk object通过标准Java序列化库被转换成字节流，序列化数据是更加CPU密集计算的方式，但是一般来说内存使用更为高效，例如Java序列化比完整的object更为高效
    - 当每个partition上的RAM不足以存储数据时，可以选择存储在磁盘上，这种方式在重复计算的情形下是显然变慢的，但是对于长链transformation来说容错更好，也是处理海量数据的唯一选项
- RDD class的persist方法（PySaprk中DataFrame拥有persist方法）允许使用者控制RDD以何种方式被存储
    - useMemory：如果设置，RDD将被存储到内存
    - useOffHeap：如果设置，RDD将被存储到Spark Executor之外的额外系统中
    - deserialized：如果设置，RDD将被存储为反序列化对象
    - replication：是一个整数，来控制持久化数据的副本数
- PySpark内的使用：
    - [RDD持久化](#rdd_persist)
    - [DataFrame持久化](#dataframe_persist)
##### RDD的逻辑组成
- Spark使用五个主要的属性去表示RDD
    - 其中三个必须的属性为组成RDD的partition对象列表、为每一个partition计算iterator的function、所依赖其他RDD的列表
    - 可选的，对于RDD的rows表示Scala的tuple时，有一个分区器（partitioner）
    - 可选的，一个优先位置的列表（对于HDFS file）
- 不可变性及相关RDD接口
    - partitions() 返回组成分布式数据集的partition对象列表
    - iterator(p, parentIters) p为要计算得到的partition，parentIters为每个父partitions，该方法旨在被安一定顺序调用来计算得到该RDD的所有partitions。并非设计来被用户直接调用
    - dependencies() 返回依赖对象的列表，区分narrow dependencies和wide depies（shuffle）
    - 如果在元素与分区之间存在关联时将返回partitioner对象的类型，如hashPartitioner
        - 所谓的关联指并非表示为key/value（tuple）的对象
    - perferredLocations(p) 返回关于partition p的数据本地化信息。特别的，该方法返回能代表p被分布存储的每个node的字符串列表，如RDD表示HDFS file时，每个字符串代表分区被存储的节点的hadoop name
#### DataFrames
- 是一种不可变的分布式数据库，这种数据集被组织成指定的列，类似于关系型数据库的表
- 通过在分布式数据集上施加结构，让Spark用户利用Spark SQL来查询结构化数据或者使用Spark表达式方法

##### DataFrames支持的格式
- JSON
- JDBC
    - DB2、Derby、MsSQL、MySQL、Oracle、Postgres
- Parquet（TODO）
    - 是面向分析型业务的列式存储格式，空间效率高
- Hive tables

#### Spark_Streaming
- 是一种高级API，内置一系列receiver，接收很多来源的流数据，如Apache Kafka、Flume、Twitter等，Spark Streaming将数据以500ms到更大的间隔窗口分割为较小的batch（称为DStream，DStream建立在RDD上），交付给Saprk Engine处理，生成batch结果集
    - 使用场景
        - 流数据ETL：在数据进入下流系统前进行数据清洗和聚合
        - 触发器：实时监测行为或者异常，触发下游动作
        - 数据浓缩：结合流数据和其他数据集，进行更为丰富的分析
        - 复杂会话和持续学习：持续更新机器学习模型
#### Job_Stage_Task
![app_tree](/assets/app_tree.png)
- Job
    - job间顺序执行
    - 在遇到action 算子（即操作，对比于转换）的时候会提交一个Job
- Stage
    - stage间顺序执行
    - 在tranform算子（即转换，对比于操作）中，会分为两种情况
        - 如果子partition拥有明确的父partition而与父partitions中的数据无关，即为narrow dependency（如map、filter、mapPartitions、flatMap）  
        ![narrow_dp](/assets/narrow_dp.png)
        - 如果子partition并不拥有明确的父partition，以来的partition取决于父partition中存在的值，即为wide dependency。也称为shuffle依赖（如groupByKey、reduceByKey、sortByKey）
        ![wide_dp](/assets/wide_dp.png)
        - join相对比较复杂，因为该操作究竟是narrow dependency还是wide dependency取决于两个父RRDs是如何分区的
    - job的stage划分便是根据shuffle依赖进行的，为两个stage的分界
- Task
    - 代表了stage的并行度
    - 在计算的时候，每个分区会起一个task，所以RDD分区数目决定了task数目
- DAG
    - Spark的高阶调度层使用RDD之间的依赖来为每一个Job建立Stages的Graph
#### DataSet
- 旨在提供给用户一种轻松转换对象的方式，由于其强制类型指定及python弱类型的特性加上pythonic的规范，python暂不支持

### Catalyst优化器
优化逻辑,接收query plan，然后将其转换为执行计划，使得Spark SQL执行快速
![Cost_Optimizer](/assets/Cost_Optimizer.png)

### Spark_job调度
- 一个Spark cluster可能并发执行多个Spark Applications
- 同时，一个Spark Application也会并发执行多个job

#### Applications间资源分配
- 静态分配，每个应用被分配有限的最大使用资源
- 动态分配，根据需要增加和去除executor，借助启发式资源预估

#### Spark应用分配过程
![job_schedule](/assets/job_schedule.png)
图中表示了当SparkContext start时的情形。首先driver程序ping cluster manager，cluster manager在nodes中启动一些Spark executors（图中黑色方形），一个node可以拥有多个executor，但是一个executor不能分布在不同node（蓝色圈圈）。一个RDD将被评估分布在不同的partition（红色椭圆），每一个executor可以有多个partition，但是一个partition不能跨越多个executor

- 默认地，Spark以先进先出调度jobs。然而Spark也提供了一种公平的调度方式，为每一个job分配一些tasks直到所有的job完成，这保证了每一个job都能获得集群资源的一部分。随后Spark application按顺序启动任务，然后actions被在SparkContext上触发

#### RRD vs DataFrame
对于DataFrame来说，其实也是基于RDD的，其通过Catalyst优化器最终的物理计划实质是操作RDD

##### 什么时候使用RDD
- 想要使用低阶transformation或者actions来控制你的数据集
- 你的数据是非结构化的
- 想要使用函数化编程模型，而不是域表达式
- 对于结构化或者半结构化数据，可以放弃一部分优化和效率

##### 什么时候使用DataFrame
- 想要丰富的语义、高阶抽象、指定域语法的API
- 想要使用高阶表达式，如filter、map、聚合、平均、SQL查询、列访问、lambda表达式，来操作半结构化数据
- 想要保证类型安全

### 部署

#### 问题汇总
- PySpark及SparkShell虚拟内存溢出问题
    - https://www.jianshu.com/p/62800898dd02
    - https://www.jianshu.com/p/42724e69d43b

### PySpark

#### SparkContext
SparkContext是Spark functionality的入口，一个SparkContext对象代表了与Spark集群的连接，可用来创建RDD和在指定集群广播变量，以下统一使用sc代表
```python
# 定义为class pyspark.SparkContext(master=None, appName=None, sparkHome=None, pyFiles=None, environment=None, batchSize=0, serializer=PickleSerializer(), conf=None, gateway=None, jsc=None, profiler_cls=<class 'pyspark.profiler.BasicProfiler'>)

from pyspark import SparkContext

sc = SparkContext('yarn-client', 'hello')
```

##### Master分类
- local	本地以一个worker线程运行(例如非并行的情况).
- local[K]	本地以K worker 线程 (理想情况下, K设置为你机器的CPU核数).
- local[*]	本地以本机同样核数的线程运行.
- spark://HOST:PORT	连接到指定的Spark standalone cluster master. 端口是你的master集群配置的端口，缺省值为7077.
- mesos://HOST:PORT	连接到指定的Mesos 集群. Port是你配置的mesos端口， 缺省是5050. 或者如果Mesos使用ZOoKeeper,格式为 mesos://zk://....
- yarn-client	以client模式连接到YARN cluster. 集群的位置基于HADOOP_CONF_DIR 变量找到.
- yarn-cluster	以cluster模式连接到YARN cluster. 集群的位置基于HADOOP_CONF_DIR 变量找到.

#### SparkSession
SparkSession是使用DataSet和DataFrame API编程操作Spark的入口，可被用来创建DataFrame、注册DataFrame为table、在table上执行SQL、缓存table、读取parquet（？什么意思）文件等，以下统一使用spark
```python
# 定义为class pyspark.sql.SparkSession(sparkContext, jsparkSession=None)

import pyspark

# 通过SparkContext创建SparkSession
spark =pyspark.sql.SparkSession(sc)

# 利用builder
# getOrCreate()获取或者创建一个基于在builder中设置的option的新的SparkSession
spark = SparkSession.builder \
     .master("local") \
     .appName("Word Count") \
     .config("spark.some.config.option", "some-value") \
     .getOrCreate()
```

#### RDD

##### RDD function（以下rdd代表Rdd对象）
- RDD内部机制
    - RDD转换通常是惰性的，意味着任何转换仅在调用数据集操作时才执行，这有利于Spark优化执行
    - 每个转换并行执行
    - 由于Py4J与JVM之间的通信开销，Python RDD比Scala慢很多
- 转换
    - rdd.map(f, preservesPartitioning=False)
        - 应用在每一个RDD元素上，返回新的Rdd对象
    - rdd.filter(f, preservesPartitioning=False)
        - 从数据集中选择符合条件的元素
    - rdd.flatMap(f, preservesPartitioning=False)
        - 应用在每一个RDD元素上，返回新的Rdd对象，相较map来说返回的是扁平的结果
    - rdd.distinct(numPartitions=None)
        - 从数据集中得到不重复的元素，返回新的Rdd对象
        - TIP！本操作是高开销的操作，仅在必要时使用
    - rdd.sample(withReplacement, fraction, seed=None)
        - 随机取样，返回新的Rdd对象
        - 参数
            - withReplacement：布尔参数，表示元素是否可以被取样多次（即是否取出后放回）
            - fraction：取样数期望
                - 当withReplacement为false，则fraction的取值范围为[0, 1],代表每个元素都有fraction的几率被取样，最终样本数量可能是0~len(rdd)
                - 当withReplacement为true,则fraction取值为[0, max_int]，代表期望每个元素被取样多次，最终样本数量为len(rdd) * fraction
            - seed：随机数种子
    - rdd.leftOuterJoin(other, numPartitions=None)
        - 左外连接，返回新的Rdd对象
        - TIP！本操作是高开销的操作，仅在必要时使用
        - 示例：
        ```python
        rdd1 = sc.parallelize([('a', 1), ('b', 4), ('c',10)])
        rdd2 = sc.parallelize([('a', 4), ('a', 1), ('b', '6'), ('d', 15)])
        rdd3 = rdd1.leftOuterJoin(rdd2)
        # [('c', (10, None)), ('b', (4, '6')), ('a', (1, 4)), ('a', (1, 1))]

        rdd4 = rdd1.join(rdd2)
        # [('b', (4, '6')), ('a', (1, 4)), ('a', (1, 1))]

        rdd5 = rdd1.intersection(rdd2)
        # [('a', 1)]
        ```
    - rdd.repartition(numPartitions)
        - 数据重新分区
        - TIP！本操作会重组数据，影响性能
- 操作
    - rdd.take(num)
        - 获取rdd前num个元素
    - rdd.takeSample(withReplacement, num, seed=None)
        - 获取rdd数据的固定大小的子集
        - 参数
            - withReplacement：布尔参数，表示元素是否可以被取样多次（即是否取出后放回）
            - num：要返回的数据数量
            - seed：随机数种子
    - rdd.collect()
        - 返回rdd所有数据
        - TIP！注意性能损耗
    - rdd.reduce(f)
        - 使用指定方法聚合数据
        - TIP！需要注意的是，调用reduce的rdd中的数据需要满足以下条件
            - 元素顺序改变，结果不变
            - 如当f为对两个参数执行除法运算则不满足，结果并非所预期
    - rdd.reduceByKey()
        - 针对不同key执行reduce
    - rdd.count() & rdd.countByKey()
    - rdd.saveAsTextFile(path, compressionCodecClass=None)
        - 将rdd数据存储至文本文件中，可使用sc.textFile(path)读取
    - rdd.foreach(f)
        - 对rdd中的每一个元素执行f

##### RDD_Persist
RDD persist与DataFrame persist执行方式及拥有方法一致，唯一的不同是cache和persist的默认值为MEMORY_ONLY

#### DataFrame

##### 创建DataFrame
```python
# 生成RDD
stringJSONRDD = sc.parallelize(("""
  { "id": "123",
"name": "Katie",
"age": 19,
"eyeColor": "brown"
  }""",
"""{
"id": "234","name": "Michael",
"age": 22,
"eyeColor": "green"
  }""", 
"""{
"id": "345",
"name": "Simone",
"age": 23,
"eyeColor": "blue"
  }"""))

# 创建DataFrame
swimmersJSON = spark.read.json(stringJSONRDD)

# 创建临时表
swimmersJSON.createOrReplaceTempView("swimmersJSON")
```

##### 查询DataFrame
```python
# show(num) 打印前num行数据，默认为10
swimmersJSON.show()

# 筛选 swimmers为DataFrame对象
swimmers.select("id", "age").filter("age = 22").show()
# 与上述同效果
swimmers.select(swimmers.id, swimmers.age).filter(swimmers.age == 22).show()
# like
swimmers.select("name", "eyeColor").filter("eyeColor like 'b%'").show()

# SQL方式
spark.sql("select * from swimmersJSON").collect()
```

##### schema定义
- 自动生成
    - 通过反射来实现schema自动生成，该反射不是python reflection，而是参照模式反射（schema reflection）
- 指定schema
    - 示例
    ```python
    # Import types
    from pyspark.sql.types import *

    # 生成RDD
    stringCSVRDD = sc.parallelize([
    (123, 'Katie', 19, 'brown'), 
    (234, 'Michael', 22, 'green'), 
    (345, 'Simone', 23, 'blue')
    ])

    # 声明schema
    # structField的参数分别为：
    # name: 字段名
    # dataType: 数据类型
    # nullable: 是否可以为空
    schema = StructType([
    StructField("id", LongType(), True),    
    StructField("name", StringType(), True),
    StructField("age", LongType(), True),
    StructField("eyeColor", StringType(), True)
    ])

    # 利用RDD和schema创建DataFrame
    swimmers = spark.createDataFrame(stringCSVRDD, schema)
    # 通过DataFrame创建临时表
    swimmers.createOrReplaceTempView("swimmers")
    ```

##### DataFrame_persist
```python
# spark.persist函数定义
# persist(storageLevel=StorageLevel(True, True, False, False, 1))
# 默认值为 StorageLevel.MEMORY_AND_DISK

# StorageLevel class 定义
# class pyspark.StorageLevel(useDisk, useMemory, useOffHeap, deserialized, replication=1)
# 因为python为弱类型，deserialized应当始终为False

# StorageLevel预置常量
# StorageLevel.DISK_ONLY = StorageLevel(True, False, False, False)
#StorageLevel.DISK_ONLY_2 = StorageLevel(True, False, False, False, 2)
#StorageLevel.MEMORY_ONLY = StorageLevel(False, True, False, False)
#StorageLevel.MEMORY_ONLY_2 = StorageLevel(False, True, False, False, 2)
#StorageLevel.MEMORY_AND_DISK = StorageLevel(True, True, False, False)
#StorageLevel.MEMORY_AND_DISK_2 = StorageLevel(True, True, False, False, 2)
#StorageLevel.OFF_HEAP = StorageLevel(True, True, True, False, 1)

from pyspark.storagelevel import StorageLevel

# 等价于spark.cache()
spark.persist(StorageLevel.MEMORY_AND_DISK)
```

##### 数据预处理
- 重复数据、缺失值、离群值检测

#### SparkStreaming
##### 非全局聚合（无状态流处理）
```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

# 以两个工作线程方式创建SparkContext
sc = SparkContext("local[2]", "NetworkWordCount")

# 以1s间隔创建SparkStreamingContext
ssc = StreamingContext(sc, 1)

# 创建连接到9999端口的DStream
lines = ssc.socketTextStream("localhost", 9999)

# 分割数据
words = lines.flatMap(lambda line: line.split(" "))

# 在每一个batch计算count
pairs = words.map(lambda word: (word, 1))
wordCounts = pairs.reduceByKey(lambda x, y: x + y)

# 打印在这个DStream中每一个RDD的前十个元素DStream 
wordCounts.pprint()

# 开启流处理
ssc.start()             
# 等待中断
ssc.awaitTermination() 

# 运行前执行nc -lk 9999监听9999端口并在运行后通过stdin获取输入
```

- 为什么是local[2],local是否可以
    - 当在本地运行Spark streaming程序时，不要使用"local"或者"local[1]"作为master URL。它们都意味着只有一个线程会在运行本地程序时被使用。如果你基于receiver（如：sockets、kafka、flume的receiver）创建input DStream，那么单独的线程将被用于运行receiver，导致并没有多余的线程来处理接收到的数据，因而要使用"local[n]"作为master URL，并且n大于receivers的数量
    - 对于集群运行该逻辑来说，分配给Spark Streaming应用的内核数应当大于receivers的数量
- 使用多Receiver情形
    - 在从网络（sockets、kafka、flume等）获取数据时，需要反序列化数据并存储于Spark，如果接收数据成为系统中的瓶颈，那么可以考虑并行接收数据，如接受kafka数据时，可以将同时接受多个topic的receiver拆分为多个，这样会有多个DStream流，每个DStream会创建一个receiver
- 如果在receiver在9999端口开启之前开启
    - 每个时间点会出现连接失败错误，即每次操作都尝试新建连接，实际上，当SparkStreaming每次时间间隔到了进行提交作业时，都会首先start Receiver，本次Job运行完成时再stop Receiver，如下是一个时间点在没有开启9999的一个情况

##### 全局聚合(有状态的流处理)
- 如果在处理流数据时，每次的输出是迄今为止数据的统计值，那么极有可能计算该值的时间超过了batch interval，这样意味着我们会不断地在后台运行试图跟上数据流的速度
    - 对于任何一个batch的数据，将其转换为DataFrame，并将数据插入持久化表，随着调度延迟的不断增加，性能会越来越低
    - 可以通过mapWithState创建全局聚合

- 修改上一节为全局聚合如下

```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

# 以两个工作线程方式创建SparkContext
sc = SparkContext("local[2]", "NetworkWordCount")

# 以1s间隔创建SparkStreamingContext
ssc = StreamingContext(sc, 1)

# 创建检查点以提供容错
ssc.checkpoint("checkpoint")

# 创建连接到9999端口的DStream
lines = ssc.socketTextStream("localhost", 9999)

 # 定义统计值更新函数
def updateFunc(new_values, last_sum):
   return sum(new_values) + (last_sum or 0)

# 分割数据并通过updateStateByKey更新全局聚合数据
running_counts = lines.flatMap(lambda line: line.split(" "))\
           .map(lambda word: (word, 1))\
           .updateStateByKey(updateFunc)

# 打印在这个DStream中每一个RDD的前十个元素DStream 
running_counts.pprint()

# 开启流处理
ssc.start()             
# 等待中断
ssc.awaitTermination() 

# 运行前执行nc -lk 9999监听9999端口并在运行后通过stdin获取输入
```

- 关于updateStateByKey与mapWithState
    - 性能
        - updateStateByKey的性能与所维护的state信息大小相关
        - mapWithState的性能与batch的大小有关
    - 在PySpark中暂未有mapWithState（黑人问号）
    - 进一步探究底层实现方式 TODO

##### 结构化流处理
- 旨在通过将Streaming与DataSet/DataFrame结合

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import explode
from pyspark.sql.functions import split

# 创建Spark Session
spark = SparkSession \
   .builder \
   .appName("StructuredNetworkWordCount") \
   .getOrCreate()

# 设置分区数，见如下问题
spark.conf.set("spark.sql.shuffle.partitions", "8")

# 创建读取输入batch的DataFrame
lines = spark\
   .readStream\
   .format('socket')\
   .option('host', 'localhost')\
   .option('port', 9999)\
   .load()

# 分割words
words = lines.select(
  explode(
         split(lines.value, ' ')
  ).alias('word')
)

# 计算
wordCounts = words.groupBy('word').count()

# 计算并输出到console
query = wordCounts\
    .writeStream\
    .outputMode('complete')\
    .format('console')\
    .start()

# 等待流计算结束
query.awaitTermination()
```

- 问题
    - 再执行上述代码时，发现效率反而不如不使用DataFrame，每个stage运行高达10s上下，经排查发现，该job的部分stage产生了200个task，应该是spark.sql.shuffle.partitions的默认值，导致很小的任务却处理速度极慢，故修改该值
        - spark.conf.set("spark.sql.shuffle.partitions", "8")

#### Spark-UDF
- UDF（User Defined Function），Spark Python API支持UDF，这些用户定义函数一次操作一行数据，因此产生了很高的序列化和调用开销，导致为了提高性能，采取Scala或者Java来定义UDF然后通过python来调用
- 基于Apache Arrow构建的Pandas UDFs使得能够完全通过python创建低开销、高性能的UDFs
- 在Spark2.3，有两种类型的Pandas UDFs：scalar和grouped map，2.4新增了grouped aggregate

##### 原有udf方式
```python
from pyspark import SparkContext
from pyspark.sql import SparkSession
from pyspark.sql.functions import udf

sc = SparkContext('yarn-client', 'hello')
rdd = sc.parallelize(
    ("""{"id":1, "v":1}""", """{"id":2, "v":100}""")
)
spark = SparkSession.builder.config("spark.sql.shuffle.partitions", "8").getOrCreate()
lines = spark.read.json(rdd)

lines.createOrReplaceTempView("insJSON")

# 返回类型需要严格限制，即便是integer和double也不可以混用（python的弱类型与java/scala的强类型）
@udf('integer')
def plus_one(v):
    return v+1

# withColumn(colName, col) 返回新的DataFrame通过新增新的列或者替代原有同名的列
newlines = lines.withColumn("v2", plus_one(lines.v))

newlines.show()
```

##### pandas_udf-Scalar
pandas_udf装饰器使得plus_one函数接受一个pandas.Series参数，也返回一个pandas.Series对象
- plus_one

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import pandas_udf, PandasUDFType

spark = SparkSession.builder.appName("hello").config("spark.sql.shuffle.partitions", "8").getOrCreate()
lines = spark.createDataFrame([{"id":"1", "v":1}, {"id":"2", "v": 2}])

# 返回类型需要严格限制，即便是integer和double也不可以混用（python的弱类型与java/scala的强类型）
@pandas_udf('integer', PandasUDFType.SCALAR)
def plus_one(v):
    return v+1

# withColumn(colName, col) 返回新的DataFrame通过新增新的列或者替代原有同名的列
newlines = lines.withColumn("v2", plus_one(lines.v))

newlines.show()
```
- Cumulative Probability

```python
import pandas as pd
from scipy import stats
from pyspark.sql import SparkSession
from pyspark.sql.functions import pandas_udf, PandasUDFType

spark = SparkSession.builder.appName("hello").config("spark.sql.shuffle.partitions", "8").getOrCreate()
lines = spark.createDataFrame([{"id":"1", "v":1}, {"id":"2", "v": 2}])

@pandas_udf('double')
def cdf(v):
    return pd.Series(stats.norm.cdf(v))


newlines = lines.withColumn('cumulative_probability', cdf(lines.v))

newlines.show()
```

##### pandas_udf-grouped-map
对某个组的所有元素（map）执行某个操作

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import pandas_udf, PandasUDFType

spark = SparkSession.builder.appName("hello").config("spark.sql.shuffle.partitions", "8").getOrCreate()
lines = spark.createDataFrame([{"id":"1", "v":1}, {"id":"2", "v": 2}, {"id":"2", "v": 4}])

# 返回类型需要严格限制，即便是integer和double也不可以混用（python的弱类型与java/scala的强类型）
@pandas_udf(lines.schema, PandasUDFType.GROUPED_MAP)
# Input/output are both a pandas.DataFrame
def subtract_mean(pdf):
    return pdf.assign(v=pdf.v - pdf.v.mean())

newlines = lines.groupby('id').apply(subtract_mean)
newlines.show()
```

##### pandas_udf-grouped-aggregate
定义了聚合一到多个pandas.Series到一个Scalar值的装饰器

```python
from pyspark.sql import SparkSession, Window
from pyspark.sql.functions import pandas_udf, PandasUDFType

spark = SparkSession.builder.appName("hello").config("spark.sql.shuffle.partitions", "8").getOrCreate()

df = spark.createDataFrame(
    [(1, 1.0), (1, 2.0), (2, 3.0), (2, 5.0), (2, 10.0)],
    ("id", "v"))

@pandas_udf("double", PandasUDFType.GROUPED_AGG)
def mean_udf(v):
    return v.mean()

df.groupby("id").agg(mean_udf(df['v'])).show()
# +---+-----------+
# | id|mean_udf(v)|
# +---+-----------+
# |  1|        1.5|
# |  2|        6.0|
# +---+-----------+

w = Window \
    .partitionBy('id') \
    .rowsBetween(Window.unboundedPreceding, Window.unboundedFollowing)
df.withColumn('mean_v', mean_udf(df['v']).over(w)).show()
# +---+----+------+
# | id|   v|mean_v|
# +---+----+------+
# |  1| 1.0|   1.5|
# |  1| 2.0|   1.5|
# |  2| 3.0|   6.0|
# |  2| 5.0|   6.0|
# |  2|10.0|   6.0|
# +---+----+------+
```


### 学习过程的其他疑问question
- Spark与MapReduce本质算法区别？ TODO
- PySpark性能缺陷
    - PySpark与RDD通信模型![PySpark_RDD_communicate](/assets/PySpark_RDD_communicate.png)
    - PySpark task是如何分发的？ TODO
    - 什么影响了PySpark的性能
        - 数据从Spark worker序列化并且管道传输到Python Worker，设计JVM和Py4J上下文切换
        - 双重序列化使得任何事情都是昂贵的
        - Python Worker的启动耗费时间
        - Python 内存不在JVM中管理，使得如果利用YARN或者其他类似框架时可能会超出容器资源的限制
- DataFrame相比RDD的性能优势？
    - PySpark速度较慢的原因在于Python子进程与JVM之间的通信，对于Python DataFrame，我们有一个在Scala DataFrame周围的python包装器，避免了通信开销
        - 个人理解：包装意味着真实执行动作的还是Scala，python不再与JVM交互，而是与Scala交互（通信）
    - 利用Catalyst和Tungsten增强性能

## ref
- 《PySpark实战指南：利用Python和Spark构建数据密集型应用并规模化部署》英文原文《Learning PySpark》
    - 部分翻译感觉不太走心，参照英文阅读
- 《High Performance Spark》
    - 原理很详实，虽然没有中文版，很不错
- [PySpark 官方Api Doc](http://spark.apache.org/docs/latest/api/python/pyspark.html)
- [Sampling With Replacement and Sampling Without Replacement](https://web.ma.utexas.edu/users/parker/sampling/repl.htm)
- [sample、takeSample源码实例详解](https://blog.csdn.net/leen0304/article/details/78818743)
- [提升PySpark性能超过JVM](https://www.slideshare.net/hkarau/improving-pyspark-performance-spark-performance-beyond-the-jvm)
- [Spark应用提交指南](https://colobu.com/2014/12/09/spark-submitting-applications/)
- [运行Spark Streaming的NetworkWordCount实例](https://bit1129.iteye.com/blog/2174751)
- [introducing vectorized udfs for pyspark](https://databricks.com/blog/2017/10/30/introducing-vectorized-udfs-for-pyspark.html)
- [pyspark-pandas-udf spark官方文档](https://spark.apache.org/docs/latest/sql-pyspark-pandas-with-arrow.html)
- [A Tale of Three Apache Spark APIs: RDDs vs DataFrames and Datasets](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html)

## 相关阅读
- [深入了解Spark SQL Catalyst优化器](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)
- [美团技术-Spark](https://tech.meituan.com/tags/spark.html)