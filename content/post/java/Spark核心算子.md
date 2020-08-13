---
title: "Spark核心算子"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["Spark", "数据仓库"]
categories: ["Spark"]
author: "ChavinKing"
---

| **Spark RDD**：                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Transformation**                                           | **Meaning**                                                  |
| map(func)                                                    | 返回一个新的分布式数据集，该数据集是通过将源的每个元素传递给函数func处理形成的。 |
| filter(func)                                                 | 返回一个新的数据集，该数据集是通过func处理后在其上返回true 的源元素形成的。 |
| flatMap(func)                                                | 与map相似，但是每个输入项都可以映射成0个或多个输出项（因此func应该返回Seq而不是单个项）。 |
| mapPartitions(func)                                          | 与map相似，但是分别在每个RDD的分区（块）上运行，因此当运行在类型为T的RDD上时函数func必须被声明成 Iterator<T> => Iterator<U>（即对RDD中的每个分区的迭代器进行操作）迭代器。 |
| mapPartitionsWithIndex(func)                                 | 与mapPartitions类似，但它还为func提供表示分区索引的整数值，因此当在类型T的RDD上运行时，func的类型必须为 (Int, Iterator<T>) => Iterator<U>（即带索引的迭代器类型）。 |
| union(otherDataset)                                          | 返回一个新的数据集，其中包含了源数据集以及参数数据集中的每个元素。 |
| intersection(otherDataset)                                   | 返回一个新的RDD，其中包含源数据集中和参数数据集交集的元素。  |
| distinct([numPartitions]))                                   | 返回一个新的数据集，其中包含源数据集的所有不同元素。         |
| groupByKey([numPartitions])                                  | 在类型为(K,V)对的数据集上调用时，返回(K，Iterable <V>)对的数据集。 注意：如果要分组以便对每个键执行汇总（例如求和或平均值），则使用reduceByKey或aggregateByKey将产生更好的性能。 注意：默认情况下，输出中的并行度取决于父RDD的分区数。您可以传递一个可选numPartitions参数来设置不同数量的任务。 |
| reduceByKey(func, [numPartitions])                           | 当在(K，V)对的数据集上调用时，返回（K，V）对的数据集，其中每个键的值使用给定的reduce函数func进行汇总，该函数必须为(V，V）=> V.与groupByKey一样，reduce任务的数量可以通过可选的第二个参数配置。 |
| aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])    | 在（K，V）对的数据集上调用时，返回（K，U）对的数据集，其中每个键的值使用给定的Combine函数和中性的“零”值进行汇总。允许与输入值类型不同的聚合值类型，同时避免不必要的分配。像in中一样groupByKey，reduce任务的数量可以通过可选的第二个参数进行配置。 |
| sortByKey([ascending], [numPartitions])                      | 当在一个(K,V)对数据集上调用时，返回一个按照key值排序的(K,V)对数据集。 |
| join(otherDataset, [numPartitions])                          | 当在(K,V)和(K,W)类型的数据集上调用时，返回类型(K,(V,W))对的数据集，其中每个键都包含所有参与value成对的元素。外连接通过支持leftOuterJoin，rightOuterJoin和fullOuterJoin支持。 |
| cogroup(otherDataset, [numPartitions])                       | 当在(K,V)和(K,W)类型的数据集上调用时，返回(K,（Iterable <V>，Iterable <W>))元组的数据集。此操作也称为groupWith。 |
| cartesian(otherDataset)                                      | 在类型T和U的数据集上调用时，返回（T，U）对（所有元素对）的数据集（笛卡尔积）。 |
| coalesce(numPartitions)                                      | 将RDD中的分区数减少到numPartitions。筛选大型数据集后，对于更有效地运行操作很有用。 |
| repartition(numPartitions)                                   | 随机地重新随机排列RDD中的数据以创建更多或更少的分区，并在整个分区之间保持平衡。这总是会通过网络重新整理所有数据。 |
| repartitionAndSortWithinPartitions(partitioner)              | 根据给定的分区程序对RDD进行重新分区，并在每个结果分区中，按其键对记录进行排序。这比repartition在每个分区内调用然后排序更为有效，因为它可以将排序推入洗牌机制。 |
| *sample(withReplacement, fraction, seed)*                    | *Sample a fraction fraction of the data, with or without replacement, using a given random number generator seed.* |
| *pipe(command, [envVars])*                                   | *通过外壳命令（例如Perl或bash脚本）通过管道传递RDD的每个分区。将RDD元素写入进程的stdin，并将输出到其stdout的行作为字符串的RDD返回。* |
|                                                              |                                                              |
|                                                              |                                                              |
| **Action**                                                   | **Meaning**                                                  |
| reduce(func)                                                 | 使用函数func（该函数接受两个参数并返回一个）来聚合数据集的元素。该函数应该是可交换的和关联的，以便可以并行正确地计算它。 |
| collect()                                                    | 在驱动程序中将数据集的所有元素作为数组返回。这通常在返回足够小的数据子集的过滤器或其他操作之后很有用。 |
| count()                                                      | 返回数据集中的元素数。                                       |
| first()                                                      | 返回数据集的第一个元素（类似于take（1））。                  |
| take(n)                                                      | 返回具有数据集的前n个元素的数组。                            |
| takeOrdered(n, [ordering])                                   | 使用自然顺序或自定义比较器返回RDD 的前n个元素。              |
| saveAsTextFile(path)                                         | 将数据集的元素以文本文件（或文本文件集）的形式写入本地文件系统，HDFS或任何其他Hadoop支持的文件系统中的给定目录中。Spark将在每个元素上调用toString，以将其转换为文件中的一行文本。 |
| saveAsSequenceFile(path)                                     | 将数据集的元素作为Hadoop SequenceFile写入本地文件系统，HDFS或任何其他Hadoop支持的文件系统中的给定路径中。这在实现Hadoop的Writable接口的键/值对的RDD上可用。在Scala中，它也可用于隐式转换为Writable的类型（Spark包括对基本类型（如Int，Double，String等）的转换）。 |
| saveAsObjectFile(path)                                       | 使用Java序列化以简单的格式编写数据集的元素，然后可以使用 SparkContext.objectFile()进行加载。 |
| countByKey()                                                 | 仅在类型（K，V）的RDD上可用。返回（K，Int）对，包含每个键的计数。 |
| foreach(func)                                                | 在数据集的每个元素上运行函数func。通常这样做是出于副作用，例如更新累加器或与外部存储系统交互。 |
| *takeSample(withReplacement, num, [seed])*                   | *返回带有数据集num个元素的随机样本的数组，带有或不带有替换，可以选择预先指定一个随机数生成器种子。* |
|                                                              |                                                              |
|                                                              |                                                              |
| **Spark Streaming****：**                                    |                                                              |
| **Transformation**                                           | **Meaning**                                                  |
| map(func)                                                    | 通过将源DStream的每个元素传递给函数func来返回新的DStream 。  |
| flatMap(func)                                                | 与map相似，但是每个输入项可以映射到0个或多个输出项。         |
| filter(func)                                                 | 通过仅选择func返回true 的源DStream的记录来返回新的DStream 。 |
| repartition(numPartitions)                                   | 通过创建更多或更少的分区来更改此DStream中的并行度。          |
| union(otherStream)                                           | 返回一个新的DStream，其中包含源DStream和otherDStream中的元素的并集。 |
| count()                                                      | 通过计算源DStream的每个RDD中的元素数，返回一个新的单元素RDD DStream。 |
| reduce(func)                                                 | 通过使用函数func（带有两个参数并返回一个）来聚合源DStream的每个RDD中的元素，从而返回一个单元素RDD的新DStream 。该函数应具有关联性和可交换性，以便可以并行计算。 |
| countByValue()                                               | 在元素类型为K的DStream上调用时，返回一个新的（K，Long）对的DStream，其中每个键的值是其在源DStream的每个RDD中的频率。 |
| reduceByKey(func, [numTasks])                                | 在（K，V）对的DStream上调用时，返回一个新的（K，V）对的DStream，其中使用给定的reduce函数聚合每个键的值。注意：默认情况下，这使用Spark的默认并行任务数（本地模式为2，而在集群模式下，此数量由config属性确定spark.default.parallelism）进行分组。您可以传递一个可选numTasks参数来设置不同数量的任务。 |
| join(otherStream, [numTasks])                                | 在（K，V）和（K，W）对的两个DStream上调用时，返回一个新的（K，（V，W））对的DStream，其中每个键都有所有元素对。 |
| cogroup(otherStream, [numTasks])                             | 在（K，V）和（K，W）对的DStream上调用时，返回一个新的（K，Seq [V]，Seq [W]）元组的DStream。 |
| **transform(func)**                                          | **通过对源DStream应用任意的RDD-to-RDD函数来返回新的DStream。这可用于在DStream上执行任意RDD操作。** |
| **updateStateByKey(func)**                                   | **返回一个新的“状态” DStream，在该DStream中，通过在键的先前状态和键的新值上应用给定函数来更新每个键的状态。这可用于维护每个键的任意状态数据。** |
|                                                              |                                                              |
|                                                              |                                                              |
| **Transformation**                                           | **Meaning**                                                  |
| window(windowLength, slideInterval)                          | 返回基于源DStream的窗口批处理计算的新DStream。               |
| countByWindow(windowLength, slideInterval)                   | 返回流中元素的滑动窗口计数。                                 |
| reduceByWindow(func, windowLength, slideInterval)            | 返回一个新的单元素流，该流是通过使用func在滑动间隔内聚合流中的元素而创建的。该函数应该是关联的和可交换的，以便可以并行正确地计算它。 |
| reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks]) | 当在（K，V）对的DStream上调用时，返回新的（K，V）对的DStream，其中使用给定的reduce函数func 在滑动窗口中的批处理上聚合每个键的值。注意：默认情况下，这使用Spark的默认并行任务数（本地模式为2，而在集群模式下，此数量由config属性确定spark.default.parallelism）进行分组。您可以传递一个可选 numTasks参数来设置不同数量的任务。 |
| reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks]) | 上述方法的一种更有效的版本，reduceByKeyAndWindow()其中，使用前一个窗口的减少值递增计算每个窗口的减少值。这是通过减少进入滑动窗口的新数据并“逆向减少”离开窗口的旧数据来完成的。一个示例是在窗口滑动时“增加”和“减少”键的计数。但是，它仅适用于“可逆归约函数”，即具有相应的“逆归约”函数（作为参数invFunc）的归约函数。像in中一样reduceByKeyAndWindow，reduce任务的数量可以通过可选参数配置。请注意，必须启用检查点才能使用此操作。 |
| countByValueAndWindow(windowLength, slideInterval, [numTasks]) | 在（K，V）对的DStream上调用时，返回新的（K，Long）对的DStream，其中每个键的值是其在滑动窗口内的频率。像in中一样 reduceByKeyAndWindow，reduce任务的数量可以通过可选参数配置。 |
|                                                              |                                                              |
|                                                              |                                                              |
| **Output Operation**                                         | **Meaning**                                                  |
| print()                                                      | 在运行流应用程序的驱动程序节点上，打印DStream中每批数据的前十个元素。这对于开发和调试很有用。 |
| saveAsTextFiles(prefix, [suffix])                            | 将此DStream的内容另存为文本文件。基于产生在每批间隔的文件名的前缀和后缀："prefix-TIME_IN_MS[.suffix]"。 |
| saveAsObjectFiles(prefix, [suffix])                          | 将此DStream的内容保存为SequenceFiles序列化Java对象的内容。基于产生在每批间隔的文件名的前缀和 后缀："prefix-TIME_IN_MS[.suffix]"。 |
| saveAsHadoopFiles(prefix, [suffix])                          | 将此DStream的内容另存为Hadoop文件。基于产生在每批间隔的文件名的前缀和后缀："prefix-TIME_IN_MS[.suffix]"。 |
| foreachRDD(func)                                             | 最通用的输出运算符，将函数func应用于从流生成的每个RDD。此功能应将每个RDD中的数据推送到外部系统，例如将RDD保存到文件或通过网络将其写入数据库。请注意，函数func在运行流应用程序的驱动程序进程中执行，并且通常在其中具有RDD操作，这将强制计算流RDD。 |