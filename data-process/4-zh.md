数据处理框架
==========

有了数据处理应用，我们能够开始处理数据了，但这还不足以上线到生产环境，我们需要一个框架保证生产环境的稳定运行。

## 基本特性
- 稳定 / 容错
- 分布式
- 方便管理

现有的有以下几个选择：

## Hadoop Yarn
老牌资源管理和进程管理工具，分布式管理和运行。[Samza](https://samza.apache.org/)就是基于Hadoop Yarn运行的

## Kubernetes
TODO

## Confluent Platform
Kafka生态中推荐的自家的处理框架。
- REST API: 非常容易上手
- 完全依赖kafka
- 闭源
- 稳定欠佳
- 分布式
- kafka生态建立中，但不完整