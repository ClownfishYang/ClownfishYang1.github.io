---
layout: 'spark title: RDD'
date: 2020-11-10 15:09:30
tags:
- Spark
- 大数据
- Demo
categories:
- 技术
---
　　最近正在中心整理之前学得一些知识，正好有空整理下Spark的大部分实现，项目会放到[SparkJobDemo]()上。

### 项目构建
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

### 常见业务处理





