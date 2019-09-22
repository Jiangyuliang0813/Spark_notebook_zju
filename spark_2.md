# 第二章 Spark下载与入门
---
## 2.1 下载Spark
```py
>>> cd ~ #切换到目录下
>>> tar -xf spark-2.1.1-bin-hadoop2.7.tgz  #解压文件
>>> cd spark-2.1.1-bin-hadoop2.7  #进入文件夹
>>> ls   #展示列表内容
```
- README.md 
    包含用来入门Spark的简单的使用说明
- bin 
    包含可以用来和Spark进行各种方式交互的一系列可执行文件
- examples
    包含一些可以查看和运行的Spark程序
---
## 2.2 Spark中Python和Scala的shell
- 打开Spark shell  (python)
```py
>>> ./bin/pyspark #进入Spark目录后输入
```
- 打开Spark shell   (scala)
```py
>>> ./bin/spark-shell #进入Spark目录后输入
```
- 利用shell从本地文本文件中创建一个RDD来做一些简单的即使统计
```py
#利用python进行行数统计
>>> lines = sc.textFile("README.md")  #创建一个名字为lines的RDD

>>> lines.count() #统计RDD中的元素个数
127
>>> lines.first() #返回这个RDD中的第一个元素，也就是README.md的第一行
'#Apache Spark'
```

```scala
//利用scala进行行数统计
scala> val lines = sc.textFile("README.md") //创建一个名为lines的RDD 
lines: spark.RDD[String] = MappedRDD[...]

scala> lines.count() //统计RDD中的元素个数
res0: Long = 127

scala> lines.first() //这个RDD中第一个元素，也就是README.md的第一行
res1: String = #Apache Spark
```
---
## 2.3 Spark核心概念简介
- 每个Spark应用都用一个 ***驱动器程序***（driver program）来发起集群上的各种并行操作。驱动器程序中包含应用的main 函数，并且定义了集群上的分布式数据集，还对这些分布式数据集应用了相关操作。前面的例子，Spark shell就是驱动器程序
- 驱动器程序通过一个 ***SparkContext*** 对象来访问Spark。这个对象代表队计算集群的一个连接。shell启动时已经自动创建了一个SparkContext对象，是一个叫做sc的变量。
- 一旦有了SparkContext,就可以创建RDD。
- 驱动器程序一般要管理多个 ***执行器*** （executor）节点。
- 我们有很多传递函数的API，以“python”为例，进行筛选
```py
#python版本进行筛选
>>> lines = sc.textFile("README.md")

>>> pythonLines = lines.filter(lambda line: "Python" in line)

>>> pythonLines.first()
u'## Interactive Python Shell'
```

```scala
//scala版本进行筛选
scala> val lines = sc.textFile("README.md") // 创建一个叫lines的RDD
lines: spark.RDD[String] = MappedRDD[...]

scala> val pythonLines = lines.filter(line => line.contains("Python"))
pythonLines: spark.RDD[String] = FilteredRDD[...]

scala> pythonLines.first()
res0: String = ## Interactive Python Shell
```
---
## 2.4 独立应用
## 2.5 总结
