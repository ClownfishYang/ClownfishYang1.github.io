---
title: 'Spark项目Demo'
date: 2020-11-10 15:09:30
tags:
- Spark
- 大数据
- Demo
categories:
- 技术
---
　　整理Spark的大部分实现，项目会放到[SparkJobDemo](https://github.com/ClownfishYang/SparkJobTemplate)上，持续更新。

<!--more-->

## 项目构建
　　项目使用的是Scala+Maven，基于[Spark-Scala-Maven-Example](https://github.com/martinprobson/Spark-Scala-Maven-Example)，也可以换成[SparkGradleTemplate](https://github.com/faizanahemad/spark-gradle-template)，重点不应该放在这个上面。
　　1. 项目初始化
```
git clone https://github.com/martinprobson/Spark-Scala-Maven-Example.git
mvn -U clean install
```
    2. 单元测试
    项目集成了很多常用的功能，例如上下文初始化trait(SparkEnv)、配置文件加载(typesafe)和日志文件(grizzled)，并且继承几个简单的单元测试(SparkTest)，能够很好的检测环境问题。
```
  test("empsRDD rowcount") { spark =>
    val empsRDD = spark.sparkContext.parallelize(getInputData("/data/employees.json"), 5)
    assert(empsRDD.count === 1000)
  }

  test("titlesRDD rowcount") { spark =>
    val titlesRDD = spark.sparkContext.parallelize(getInputData("/data/titles.json"), 5)
    assert(titlesRDD.count === 1470)
  }

  private def getInputData(name: String): Seq[String] = {
    val is: InputStream = getClass.getResourceAsStream(name)
    scala.io.Source.fromInputStream(is).getLines.toSeq
  }
```

## 常见业务处理
### WordCount（词频统计）
词频统计就是统计一个或多个单词出现的次数，通过将单词转化为计数对(key-value)，即(word,count)，得到最终所有单词出现的次数。  
具体函数的选择可以参考[Spark聚合算子](./Spark聚合算子.md)。
```
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithReduce = wordPairsRDD.reduceByKey(_ + _)
```


### AvgAge（平均年龄）
平均年龄是很常见的业务需求，通过将所有年龄进行相加得到总年龄，然后再除以总人数，最终得到平均年龄，这可以测试大数据量的计算能力。
数据格式(id-age)：
```
1,18
2,54
3,47
4,11
5,26
6,8
7,12
8,3
9,24
10,31
11,32
```
案例实现
```
val ageRDD = sc.sparkContext.parallelize(readDataFile, 20)
      .map(_.split(","))
      .filter(_.length == 2)
      
// reduce 实现
// totalAge: Infinity
// totalAge: 2955154.0
//    val totalAge = ageRDD.map(_(1)).reduce(_ + _).toDouble
val totalAge = ageRDD.map(_(1).toLong).reduce(_ + _).toDouble
val count = ageRDD.count.toDouble
val avgAge = totalAge / count
info(s"totalAge: $totalAge")
info(s"count: $count")
info(s"avg age : $avgAge")

// aggregate 实现
type scoreCount = (Double, Int)
val avgAge = ageRDD.map(c => (c(1).toDouble, 1))
  .aggregate((0.0, 0))(
    (a: scoreCount, b: scoreCount) => (a._1 + b._1, a._2 + b._2),
    (a: scoreCount, b: scoreCount) => {
      ((a._1 + b._1) / (a._2 + b._2) , 1)
    }
  )._1
info(s"avg age : $avgAge")
```
案例相对简单，但如果对于海量数据场景就需要考虑一些额外的东西：
1. 整数溢出：在`reduce`实现中，若使用int进行age累加，有可能会导致溢出，换成long可以一定程度避免，但对于数据值较大或者数据量很大时还是会出现；
2. 数据过大：当partition中的数据量过大，导致超过task的内存范围，可能也会导致RPC传输的数据包过大；

这种计算通常是通过`分治`的方式进行计算，因为案例数据较简单，但大部分生产数据都会有`分类`的存在，先通过分类计算出该指标值，然后再将所有分类的指标值进行聚合计算，这就可以减少计算的数据量。案例中`aggregate`的实现就是基于此思想，先通过partition累加，partition间在进行聚合运算，最终得到avgAge。


### 人口信息指标
人口信息指标是很常见的业务需求，假设需要对某一地区的人口根据性别进行身高统计，并且计算出性别人数，性别的最高和最低身高。  
数据格式(id-sex-height)：
```
1,0,153
2,1,152
3,1,146
4,2,167
5,2,176
6,1,158
7,2,169
```
案例实现：
```
val peopleInfoRdd = sc.sparkContext.parallelize(readDataFile, 20)
  .map(_.split(","))
  .filter(_.length == 3)
  // sex - height
  .map(x => (x(1).toInt, x(2).toInt))

// 性别: 0 未知，1 男，2 女
// 数量、最低、最高、平均
type metricType = (Int, Int, Int, Long)
peopleInfoRdd.aggregateByKey((0, 0, 0, 0L))(
  (a: metricType, b: Int) => (a._1 + 1, if (a._2 == 0) b else Math.min(a._2, b), Math.max(a._3, b), a._4 + b),
  (a: metricType, b: metricType) => (a._1 + b._1, Math.min(a._2, b._2), Math.max(a._3, b._3), a._4 + b._4)
).collect()
.foreach(x =>
  info(s"sex: ${x._1}, count: ${x._2._1}, min: ${x._2._2}, max: ${x._2._3}, avg:${x._2._4 / x._2._1} ")
)
```
这里根据`sex`进行`partition`计算，能够比较方便的计算出对应的指标值，针对有些场景会通过`filter`将其分为多个RDD进行计算，相当于进行了任务拆分，这在数据较大时很常见。


### 关键词TopK
假设某搜索引擎公司要统计过去一年搜索频率最高的K个关键词或词组，为了简化问题，假设关键词组已经被整理到一个文本文件中。  
关键词或者词组可能会出现多次，且大小写格式可能不一致。
数据格式：
```
Sorting by Competition
Sorting by Competition helps you locate keywords that you may be able to rank well for with a relatively small amount of work. Generally speaking you’re looking for keywords with some Competition, as this can be an indication of market activity. The lower the Competition the easier it will be to rank well for those keywords and receive more traffic as a result.
```

案例实现：
```
// 过滤停顿词
val stopwords = Array("a","are", "to", "the","by","your","you","and","in","of","on")
sc.sparkContext.parallelize(readDataFile, 5)
      .filter(_.length > 0)
      .map(_.toLowerCase)
      .flatMap(_.split(","))
      .flatMap(_.split(" "))
      .map(_.trim)
      .filter(keyword => keyword.length > 0 && !stopwords.contains(keyword))
      .map((_, 1))
      .reduceByKey(_+_)
      // partition top K
      .repartitionAndSortWithinPartitions(new HashPartitioner(20))
      .mapPartitions(_.take(20))
      // count 降序
      .sortBy(_._2, false)
      .take(10)
      .foreach(x => info(x))

case class SearchKeywords(keyword: String, count: Int)

implicit def orderingByGradeStudentScore[A <: SearchKeywords] : Ordering[A] = {
  Ordering.by(x => -x.count)
}
```
这里通过`repartitionAndSortWithinPartitions`函数进行重分区排序，减少聚合的传输数据量，排序通过`Ordering`接口实现。