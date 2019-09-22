# 第三章 RDD编程
## 3.1 RDD基础
- Spark中的RDD就是一个不可变的分布式对象集合。每个RDD都被分为多个分区，这些分区运行在集群中的不同节点上
- 用户有两种方法创建RDD：读取一个外部数据集，或在驱动器程序里分发驱动器程序中的对象集合。
```py
#python版本进行筛选
>>> lines = sc.textFile("README.md")    
#读取文本文档创建RDD，创建完成后支持两种操作：转化操作和行动操作
 
>>> pythonLines = lines.filter(lambda line: "Python" in line)
#转化操作：会生成一个新的RDD pythonLines

>>> pythonLines.first()
#行动操作：对RDD计算出一个结果，并把结果返回到驱动器程序中，或者外部存储系统（HDFS）

u'## Interactive Python Shell'
```
- 转化操作与行动操作的区别在于Spark计算RDD的方式不同。可以在任何时候定义新的RDD，但是Spark只会 ***惰性*** 计算这些RDD。他们只有在第一次在一个行动操作中使用时，才会真正计算。
- 默认情况下，Spark的RDD会在每次对它们进行行动操作时重新计算。如果想在多个行动操作中重用同一个RDD，可以让Spark把这个RDD缓存下来。
```py
>>> pythonLines.persist
>>> pythonLines.count()
2
>>> pythonLines.first()
u'## Interactive Python Shell'
```
- 总的来说，每个Spark程序或者Shell会话都按照如下方式工作。
1. 从外部数据创建出输入RDD。
2. 使用诸如filter() 这样的转化操作对RDD 进行转化，以定义新的RDD。
3. 告诉Spark 对需要被重用的中间结果RDD 执行persist() 操作。
4. 使用行动操作（例如count() 和first() 等）来触发一次并行计算，Spark会对计算进行优化后再执行。
---
## 3.2 创建RDD
- Spark 提供了两种创建RDD 的方式：读取外部数据集，以及在驱动器程序中对一个集合进行并行化。
- 创建RDD最简单的方式就是把程序中一个已有的集合传给SparkContext的parallelize()
```py
#python
>>> lines = sc.parallelize(["pandas","i like pandas"])
```
```scala
//scala
>>> val lines = sc.parallelize(List("pandas", "i like pandas"))
```
- 更加常用的方式是从外部存储中读取数据来创建RDD（SparkContext.textFile()就是一个例子)。
```py
#python
>>> lines = sc.textFile("/path/to/README.md")
```
```scala
//scala
>>> val lines = sc.textFile("/path/to/README.md")
```
---
## 3.3 RDD操作
- 前文说到，RDD支持两种操作：***转化操作*** 和 ***行动操作*** 
- 转化操作时返回一个新RDD的操作，比如map()和filter()。
- 行动操作则是像驱动器程序返回结果或把结果写入外部系统的操作，会触发实际的计算，比如count()和first()。
- Spark对待转化操作和行动操作的方式很不一样，
### 3.3.1 转化操作
- RDD的转化操作是返回新RDD的操作。
- 许多转化操作都是针对各个元素的，也就是说，这些转化操作每次只会操作RDD 中的一个元素。不过并不是所有的转化操作都是这样的。
- 举个例子，假定我们有一个日志文件log.txt，内含有若干消息，希望选出其中的错误消息。
```py
#python
>>> inputRDD = sc.textFile("log.txt")
>>> errorsRDD = inputRDD.filter(lambda x: "error" in x)
```
```scala
//scala
>>> val inputRDD = sc.textFile("log.txt")
>>> val errorsRDD = inputRDD.filter(line => line.contains("error"))
```
- 使用另一个转化操作union()来打印出两者的行数
```py
#python
>>> errorsRDD = inputRDD.filter(lambda x: "error" in x)
>>> warningsRDD = inputRDD.filter(lambda x: "warning" in x)
>>> badLinesRDD = errorsRDD.union(warningsRDD)
#它操作了两个RDD而不是一个
```
### 3.3.2 行动操作
- 行动操作是第二种类型的RDD 操作，它们会把最终求得的结果返回到驱动器程序，或者写入外部存储系统中。
- 继续我们在前几章中用到的日志的例子，我们可能想输出关于badLinesRDD 的一些信息。为此，需要使用两个行动操作来实现：用count() 来返回计数结果，用take() 来收集RDD 中的一些元素
```py
#python
>>> print "Input had " + badLinesRDD.count() + " concerning lines"
>>> print "Here are 10 examples:"
>>> for line in badLinesRDD.take(10):
···     print(line)
```
```Scala
//Scala
>>> println("Input had " + badLinesRDD.count() + " concerning lines")
>>> println("Here are 10 examples:")
>>> badLinesRDD.take(10).foreach(println)
```
- 需要注意的是，每当我们调用一个新的行动操作时，整个RDD 都会从头开始计算。要避免这种低效的行为，用户可以将中间结果持久化。
### 3.3.3 惰性求值
- 惰性求值意味着当我们对RDD 调用转化操作（例如调用map()）时，操作不会立即执行。相反，Spark 会在内部记录下所要求执行的操作的相关信息。
- 我们不应该把RDD看作存放着特定数据的数据集，而最好把每个RDD当作我们通过转化操作构建出来的、记录如何计算数据的指令列表。把数据读取到RDD的操作也同样是惰性的。
- Spark 使用惰性求值，这样就可以把一些操作合并到一起来减少计算数据的步骤。在类似Hadoop MapReduce 的系统中，开发者常常花费大量时间考虑如何把操作组合到一起，以减少MapReduce 的周期数。而在Spark 中，写出一个非常复杂的映射并不见得能比使用很多简单的连续操作获得好很多的性能。因此，用户可以用更小的操作来组织他们的程序，这样也使这些操作更容易管理。
---
## 3.4 向Spark传递函数
### 3.4.1 Python
- 在Python 中，我们有三种方式来把函数传递给Spark。传递比较短的函数时，可以使用lambda 表达式来传递。
```py
>>> word = rdd.filter(lambda s: "error" in s)
```
- 除了lambda 表达式，我们也可以传递顶层函数或是定义的局部函数。
```py
>>> def containsError(s):
···     return "error" in s
>>> word = rdd.filter(containsError)
```
- 传递不带字段引用的Python 函数
```py
>>> class WordFunctions(object):
        ...
>>>     def getMatchesNoReference(self, rdd):
            # 安全：只把需要的字段提取到局部变量中
>>>         query = self.query
>>>         return rdd.filter(lambda x: query in x)
```
### 3.4.2 Scala
- 在Scala 中，我们可以把定义的内联函数、方法的引用或静态方法传递给Spark，就像Scala 的其他函数式API 一样。我们还要考虑其他一些细节，比如所传递的函数及其引用的数据需要是可序列化的。
- 除此以外，与Python 类似，传递一个对象的方法或者字段时，会包含对整个对象的引用。
```scala
class SearchFunctions(val query: String) {
    def isMatch(s: String): Boolean = {
        s.contains(query)
    }
    def getMatchesFunctionReference(rdd: RDD[String]): RDD[String] = {
        // 问题："isMatch"表示"this.isMatch"，因此我们要传递整个"this"
        rdd.map(isMatch)
    }
    def getMatchesFieldReference(rdd: RDD[String]): RDD[String] = {
        // 问题："query"表示"this.query"，因此我们要传递整个"this"
        rdd.map(x => x.split(query))
    }
    def getMatchesNoReference(rdd: RDD[String]): RDD[String] = {
        // 安全：只把我们需要的字段拿出来放入局部变量中
        val query_ = this.query
        rdd.map(x => x.split(query_))
    }
}
```
--- 
## 3.5 常见的转化操作和行动操作
### 3.5.1 基本RDD
1. ***针对各个元素的转化操作***
    - ***map()*** ：接受一个函数，将这个函数用于RDD中的每个元素，将函数的返回结果作为RDD对应元素的值
    - ***filter()*** ：接收一个函数，并将RDD中满足该函数的元素放入新的RDD中返回
    ```py
    #python
    nums = sc.parallelize([1, 2, 3, 4])
    squared = nums.map(lambda x: x * x).collect()
    for num in squared:
        print('{}'.format(num))
    ```
    ```scala
    //scala
    val input = sc.parallelize(List(1, 2, 3, 4))
    val result = input.map(x => x * x)
    println(result.collect().mkString(","))
    ```
    - ***flatmap()*** ：把输入的字符串切分为单词
    ```py
    #python 
    lines = sc.parallelize(["hello world", "hi"])
    words = lines.flatMap(lambda line: line.split(" "))
    words.first() # 返回"hello"
    ```
    ```scala
    //scala
    val lines = sc.parallelize(List("hello world", "hi"))
    val words = lines.flatMap(line => line.split(" "))
    words.first() // 返回"hello"
    ```
2. ***伪集合操作***
    ***RDD.distinct()*** ：生成一个只包含不同元素的新的RDD，其间使用到了(shuffle)。
    ***union(other)*** ：返回一个包含两个RDD中所有元素的RDD。（与集合概念相似，不同点在于如果输入有重复，返回数据也会包含重复）
    ***intersection(other)*** ：通过网络混洗数据来发现共有的元素
    ***subtract(other)*** ：返回一个由只存在于第一个RDD 中而不存在于第二个RDD 中的所有元素组成的RDD。
    ***cartesian(other)*** ：返回所有可能的(a,b)对，a时源RDD，b则来自于另外一个RDD。
3. ***行动操作***
    ***reduce()*** ：接收一个函数作为参数，这个函数要操作两个RDD 的元素类型的数据并返回一个同样类型的新元素。
    ```py
    #python,对rdd求和
    >>> sum = rdd.reduce(lambda x, y: x + y)
    ```
    ```scala
    //scala
    >>> val sum = rdd.reduce((x, y) => x + y)
    ```
    ***aggregate()*** ：计算RDD的平均值。
    ```py
    #python
    sumCount = nums.aggregate((0, 0),(lambda acc, value: (acc[0] + value, acc[1] + 1),(lambda acc1, acc2: (acc1[0] + acc2[0], acc1[1] + acc2[1]))))
    return sumCount[0] / float(sumCount[1])
    ```
    ```scala
    //scala
    val result = input.aggregate((0, 0))((acc, value) => (acc._1 + value, acc._2 + 1),(acc1, acc2) => (acc1._1 + acc2._1, acc1._2 + acc2._2))
    val avg = result._1 / result._2.toDouble
    ```
---
## 3.6 持久化（缓存）
- 如前所述，Spark RDD 是惰性求值的，而有时我们希望能多次使用同一个RDD。如果简单地对RDD 调用行动操作，Spark 每次都会重算RDD 以及它的所有依赖。这在迭代算法中消耗格外大，因为迭代算法常常会多次使用同一组数据。
- 为了避免多次计算同一个RDD，可以让Spark 对数据进行持久化。
- 出于不同的目的，我们可以为RDD 选择不同的持久化级别。
- RDD 还有一个方法叫作unpersist()，调用该方法可以手动把持久化的RDD 从缓
存中移除。