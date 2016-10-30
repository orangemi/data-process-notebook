kafka-数据处理
=============

## 处理拓扑
数据流处理应用由一个或多个`处理拓扑`组合而成，而每一个`处理拓扑`不外乎一下几个步骤：
![](http://docs.confluent.io/3.0.1/_images/streams-concepts-topology.jpg)http://docs.confluent.io/3.0.1/_images/streams-concepts-topology.jpg
- 获取数据
- 处理数据
- 输出数据

## 实际处理数据类型
实际上我们在处理数据时上述`处理拓扑`很少直接应用，大部分需要根据业务用以下几个方式处理数据，这些处理类型都继承于`处理拓扑`：
- 解析 parse
- 补充 join
- 过滤 filter
- 聚合 aggrenate
- 频段 window

## 难点1 如何补充数据？
数据是源源不断的增加，是一种流，在处理单条数据的时候，解析不会有任何问题，

若不是在原始数据上加以记录，要在已产生的数据流中补充数据，需要查询相关数据(表)作为补充。
查询的数据是一种关系型数据，利用某个特定的条件查询符合自己的数据。

处理完的流绝大多数情况下需要回写入关系型数据库。

如何处理`流`和`关系数据`是处理数据的难点。

### 流数据库 和 关系型数据库
kafka-streams中提出流既是流 (stream) 也是关系数据 (table)
关系数据的变化 (changelog) 就是流，而把流的每一个结果回放(replay)写入关系数据库中，将得到和原来的数据.

例如，有一张关于名字和年龄的关系表:
| Name | Age |
| ---- | --- |

当关系表的数据进行增加，改动，删除的时候可以得到他的变化流：
![](http://docs.confluent.io/3.0.1/_images/streams-table-duality-02.jpg)

利用回放可以再得到原来的数据：
![](http://docs.confluent.io/3.0.1/_images/streams-table-duality-03.jpg)

### kafka-streams
kafka-streams提供了相当优秀的API服务，利用流和关系数据的统一性处理数据：

kstream join kstream(window): 组合两个流的数据, 加入者必须是频段的（否则将是无限的）
kstream join ktable: 经过查询关系数据，将数据补充到数据流中。
ktable join ktable: 经过查询关系数据，将数据补充到关系数据中，这就是我们平时的联合查询的功能。

join的方式和我们平时用的联合查询的方式非常类似，有`left join`, `inner join`, `outer join`

### kafka compact-log
如果我们有一组关系数据，修改非常频繁，即数据的流非常多。kafka-streams是如何快速查询出他的最后的数据呢？
这需要归功于kafka数据库的功能之一：compact-log。

kafka虽然能够将数据落地到磁盘，保证数据不会因为断电或其他中断因素所影响。
但变化数据不像最终关系数据那样，保留了所有中间过程的数据。
过程数据随着时间推移，价值将变得越来越低，当磁盘遇到容量上限的时候，我们不得不删除他们。
直接删除过程数据会导致我们不能通过回放机制得到最新的关系数据，kafka在每次数据将要过期时，把过程数据打包成最终数据，但保留了最近的过程数据。
这极大的保留了数据的新鲜价值，也保证了数据可回放的能力。

### 实现原理 (WIP)

## 难点2 如何频率统计
这其实不算数据处理中的难点，而是知识缺失点。
知识来自于[Lossy Counting and Sticky Sampling](https://micvog.com/2015/07/18/frequency-counting-algorithms-over-data-streams/)

- Step 1: Divide the incoming data stream into windows.
![](https://micvog.files.wordpress.com/2015/06/step_1_lossy_counting.png?w=682&h=284)

- Step 2: Increment the frequency count of each item according to the new window values. After each window, decrement all counters by 1.
![](https://micvog.files.wordpress.com/2015/06/step_2_lossy_counting.png?w=682&h=307)

- Step 3: Repeat – Update counters and after each window, decrement all counters by 1.

- Output: The most frequently viewed items “survive”.
Given a frequency threshold f, a frequency error e, and total number of elements N, the output can be expressed as follows: Elements with count exceeding fN – eN.
Worst case we need (1/e) * log (eN) counters.

