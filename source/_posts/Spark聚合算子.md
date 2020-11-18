---
title: SparkRDD聚合算子
date: 2020-11-13 16:40:42
tags:
- Spark
- 大数据
- 聚合函数
categories:
- 技术
---
　　Spark RDD聚合算子的使用区别，持续更新。

<!--more-->

## 聚合算子

### groupByKey
　　在一个PairRDD上调用，返回一个(k,iterable<v>)，将所有相同key的pair**分组**到一个无序的集合序列中。所有的pair都加载到内存中进行存储计算，若一个key对应的value过多，则容易导致内溢出。
以WordCount为例：
```
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithGroup = wordPairsRDD.groupByKey().map(t => (t._1, t._2.sum)).collect()
```
当使用groupByKey时，通过调用[Partitioner]()将所有pair移动到key对应的计算节点，当移动的数据量大于节点的总内存时就会导致内存溢出(`MEMORY_ONLY`存储级别)。  
也就是说所有的数据都从mapTask发送到reduceTask，没有优化网络IO，只有在reductTask需要使用到所有value时才应该使用。
![](https://img.snailshub.com/images/2018/11/08/20160514090524_98484.png)
针对上面的例子，可以先计算以后在移动（预处理），而不需要移动以后在计算，这会导致大量无用的资源消耗。

### reduceByKey
　　跟`groupByKey`类似，主要的不同点是**聚合**，`reduceByKey`在调用时需要传入一个`Function`(lamdba函数)，该函数必须是类型`(V,V) => V`，在节点数据移动之前先对该节点的数据进行`Function`的处理（预处理），在数据移动之后目标节点会再次对分组的数据进行再一次`Function`处理产生最后的结果。`Function`会在mapTask和reduceTask上执行，优化了网络IO，很多聚合场景都能够使用。  
以WordCount为例：
```
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithReduce = wordPairsRDD.reduceByKey(_ + _).collect()
```
![](https://img.snailshub.com/images/2018/11/08/reduce_by.png)
针对上面的例子，`reduceByKey`通过对计算数据的预处理，减少了移动到目标节点的数据量，降低了内存溢出的风险。


### foldByKey
　　`foldByKey`用于对PairRDD进行折叠/聚合操作，跟`reduceByKey`类似，不同在于提供了初始值(zero Value)作为`Function`计算的初始值，这在部分场景下很有用。
```
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithFold = wordPairsRDD.foldByKey(0)(_ + _).collect()
```

### aggregate/aggregatebykey
　　`aggregate`和`aggregatebykey`都可以进行聚合操作，都会先对每个分区里的元素与zeroValue进行`combine`聚合(seqOp,(U, T) => U)，然后将每个分区的结果再与zeroValue进行`combine`操作(combOp,(U, U) => U)。  
　　区别在于`aggregate`对非PairRDD也能进行聚合操作，且返回的类型可以不和RDD中的类型一致，但`aggregatebykey`只能对PairRDD进行聚合操作，且返回的类型还是PairRDD，虽然key类型还是一致，但RDD中的value类型可以不一致。
```
# aggregatebykey
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithAggregate = wordPairsRDD.aggregateByKey(0)(((a, v) => a + v), ((a, v) => a + v))
# aggregate
sc.parallelize(words).aggregate(Map[String, Int]())((a,v) => {
      a + (v -> (a.getOrElse(v, 0) + 1))
    },(a,v) => {
      a ++ v.map(t => t._1 -> (t._2 + a.getOrElse(t._1, 0)))
    })

# reduce对比
val userAccesses = sc.parallelize(Array("u1", "site1"), ("u2", "site1"), ("u1", "site1"), ("u2", "site3"), ("u2", "site4"))
userAccesses.map(userSite => (userSite._1, Set(userSite._2))).reduceByKey(_++_)
userAccesses.aggregateByKey(collection.mutable.Set[String]())((set,v) => set += v, (setOne, setTwo) => setOne ++= setTwo)
```
`aggregatebykey`与`reduceByKey`的功能类似，但`aggregatebykey`支持不同的返回值类型，且提供了zeroValue能够减少计算过程中中间变量的产生，上面的对比例子使用了可变set，减少每次计算set的生成。

### combineByKey
`combineByKey(combineByKeyWithClassTag)`是通用的核心高级函数，通过一组自定义的聚合函数对PairRDD中的每个key的元素进行聚合，返回值的类型可以不和RDD中的类型一致((K,V) => C)，是大多数聚合函数的底层实现。  
提供三个自定义函数：
1. createCombiner：(V => C)，将当前key对应的V作为参数，进行类型转换并将其返回，即partition-key的初始化操作；
2. mergeValue：((C,V) => C)，将元素V合并到之前的元素C(createCombiner的返回值)上，在partition内进行，即shuffle前执行；
3. mergeCombiners：将两个C进行合并，在不同partition内进行，即shuffle后执行；

当使用`combineByKey`进行聚合操作时，若当处理的元素为新元素时(partition中首次出现的key对应的V)，使用`createCombiner`进行初始化操作；若不是新元素，说明已经初始化完成，调用`mergeValue`将其合并到C中；由于计算可能分为多个partition，最终合并需要调用`mergeCombiners`将所有结果合并。
```
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
val wordCountsWithAggregate = wordPairsRDD.combineByKey(v:Int => v, (a:Int,b:Int) => a+b,(c:Int,d:Int)=>c+d)
```


