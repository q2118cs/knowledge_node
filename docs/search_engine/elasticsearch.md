# ElasticSearch基础

ElasticSearch是一个开源的可扩展的分布式全文检索引擎，可扩展到上百台服务器，支持处理PB级数据，底层基于Lucene，使用Java进行开发。可以用于站内搜索、日志分析、数据预警及分析平台、BI系统等。

## 核心概念

- **索引(index)**：类似的数据放在一个索引，相当于一个关系型数据库。

- **类型(type)**：表示属于index种的哪种类型，相当于关系型数据库中的表。ES5.X一个index可以有多种type，6.X一个index只能有一种type，7.X逐渐移除了type。

- **映射(mapping)**：定义了每个字段的类型等信息，相当于关系型数据库中的表结构。

  | 关系型数据库   | ElasticSearch             |
  | -------------- | ------------------------- |
  | 数据库DataBase | 索引Index                 |
  | 表Table        | 索引Index类型（原为Type） |
  | 数据行Row      | 文档Document              |
  | 数据列Column   | 字段Field                 |
  | 约束Schema     | 映射Mapping               |

  

