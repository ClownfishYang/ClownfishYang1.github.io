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

