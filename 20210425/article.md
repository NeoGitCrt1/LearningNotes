####参考
##### [https://mp.weixin.qq.com/s/A3DPumB5DmbnXIV18-oavg](https://mp.weixin.qq.com/s/A3DPumB5DmbnXIV18-oavg)
https://mp.weixin.qq.com/s/SBLxyBT-tjcYfpjvTcFylA
https://mp.weixin.qq.com/s/O5ASmIRB4EHUA1kFqCGYBQ
https://my.oschina.net/u/4000872/blog/2252620
https://help.aliyun.com/document_detail/170426.html?spm=a2c4g.11186623.6.783.2ff71f86aR4SMH

####数据同步迁移手段概述
- 大数据类：为大数据产品流入数据提供服务，因为大数据产品本身特点，侧重批量定时的迁移，实时同步一般需要用特别的方法，往往和业务特征紧耦合。常见的数据迁移同步工具有 sqoop、datax 等
- 流计算类：为自身流计算框架生态服务，**侧重计算**，迁移同步更多是类似数据连接器的角色，代表的产品如 Flink
- 消息类：为自身消息产品生态服务，如丰富的 kafka connector、debezium 等
- 数据库类：数据库厂家一般都会提供原厂工具，典型如 Oracle 的 GoldenGate
- 云厂商类：云厂商提供的数据迁移同步工具，主要侧重自身云上数据库生态产品之间的互融互通和将线下自建数据库的数据上云，例如阿里云 DTS, 腾讯云 DTS ,  AWS 的 DMS 等
- 专业数据迁移同步工具: 包括部分开源产品或第三方独立公司提供的数据迁移同步工具，例如 canal、streamsets、maxwell、cloudcanal、striim、fivetran ，以及老牌数据集成厂商 Informatica 、Qlik 等所提供的产品

######优点:
- 主库更稳定：异步化解耦业务系统事务查询和复杂查询，避免复杂查询对主数据库产生影响
- 易运维、链路稳定：数据迁移同步链路有标准化产品支撑，和主业务系统、主库读写解耦。整体架构上职责清晰，易于维护和问题追踪

######缺点:
- 需要对纷繁多样的数据迁移同步工具、承载复杂查询数据库产品选型，对技术同学能力有一定要求

####MySQL 到 ES 数据实时同步

#####模型选择
1. 订阅消费
![1.png](https://upload-images.jianshu.io/upload_images/26104743-588176760e731bc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优点
- 堆积能力：由于引入了消息队列，所以整个链路是具备变更数据的堆积能力的。假设变更数据消费的比较慢，MySQL 本地较老的 binlog 文件由于磁盘空间的不足而被删除时，消息队列中的数据仍然存在，数据同步仍然可以正常进行
- 数据分发能力：引入消息队列后可以支持多方订阅。如果下游多个应用都依赖源端的变更数据，可以订阅同一份 topic 即可
- 数据加工能力：由于变更数据是由下游消费者订阅，因此订阅后可以灵活的做一些数据加工。例如从外部调用微服务接口或者反查一些数据来做数据加工都是比较方便的

缺点
- 运维成本相对较高：**整体链路较长**，端到端的数据同步链路包含了较多的组件和应用，运维要保证整体链路的稳定性需要关注更多的组件和应用
- 稳定性风险较高：由于**端到端链路较长**，整个链路中的一环出问题会导致整个端到端数据同步的稳定性。而且由于链路较长，排查和定位问题的时候也会困难很多

2. 端到端直连
![2.png](https://upload-images.jianshu.io/upload_images/26104743-bc64339ba8b184af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优点：
- 低延迟：端到端的直接同步，链路较短，延迟低
- 稳定性好：链路组件少，出问题概率较低，定位排查均比较容易。适合对数据精确性高的严苛场景。
- 功能拓展性强：对端写入消息系统，模拟订阅模式，可扩展性强
- 运维部署简单：链路组件少，部署运维更简单

缺点：
- 无

#####核心挑战
1. MySQL 关联表在 ES 上的设计
join 和 nested 类型比较
- join 适合写多读少场景，更加适合关注索引性能的场景。这意味着更新的生效会更快，但是搜索时的开销也相对大些
- nested 适合读多写少的场景，更加关注搜索的性能

2. 去规范化

2.1 主表冗余数据
业务侧将一些查询时需要的关系数据提前冗余在源表的一个字段中
- 优点：
处理模式能应对各种一对多的关联关系，对数据同步工具的功能要求低，配置简单，只需要支持单表同步到 ES 即可。
- 缺点：
索引、搜索性能非最佳：提供给 ES 的不是预构建好的宽表数据。例如例子中，订单关联的商品信息，全部存储在主表的一个object/nested/join 字段内，这种实现方式会有索引、搜索性能方面的额外开销，不是性能最佳的实现方式
业务系统侵入：业务系统写主数据的时候需要额外写入信息
主数据库表冗余过多数据：关系型数据库的表冗余了过多其他表的信息，可能存在存储和性能开销
- 总结
**不推荐**

2.2 多表订阅合并预构建宽表数据
数据同步工具同时订阅搜索时依赖的所有表，先到的数据先进到 ES，没有数据过来的字段为空。

- 优点：
数据同步工具配置同步任务较为简单，无业务入侵，不耦合业务系统逻辑
对数据同步工具要求低，除了同步以外，不需要其他额外的功能特性
基于预构建宽表的方式在 ES 上也有较好的索引和查询性能。
同步链路不会因为宽表某些列缺失数据阻塞整个数据链路的同步(是否有该优点取决于数据同步工具本身设计，如果引入时间窗口，则同步链路会因为等待列数据影响同步时效性)。
- 缺点：
基于事实表主动触发式的方式来进行宽表的构建。源端订阅的表，如果更新很少或者从来不更新产生 binlog，则对端的文档中的列值可能一直不完整，导致时效性会比较差。搜索的时候有一些列的数据会缺少
- 总结
适合构成宽表的事实表数据写入有事务保证一起落盘的场景，这样可以避免对端ES搜索到不完整的数据。
适合构建宽表不需要业务加工处理的场景，构建宽表只是单纯的将多张表的列拼接在一起，形成宽表。

宽表样例
```
{
  "mappings": {
    "_doc": {
      "properties": {
        "order_id": {
          "type": "long"
        },
        "order_price": {
          "type": "long"
        },
        "product_count": {
          "type": "long"
        },
        "discount": {
          "type": "long"
        },
        "product_id": {
          "type": "long"
        },
        "product_unit_price": {
          "type": "long"
        },
        "product_name": {
          "type": "text"
        },
      }
    }
  }
}
```

2.3 同步过程回查预构建
数据同步工具订阅的表称为主表。数据同步过程中，反查数据库查询的表称为从表。利用数据同步工具自身的能力，在订阅主表期间，自动通过回查的方式，填补宽表中的列，形成完整的宽表行数据。
- 优点：
基于反查的方式构建宽表灵活性好，可以在生成宽表前基于主表的数据对从表数据做一些轻度的数据加工
一条主表的数据，通过反查生成宽表行，可以配合数据加工生成多条宽表行数据
基于反查的方式可以比较轻松的实现跨实例的 join ，从而生成宽表行(相对好实现，具体要看数据同步工具本身是否支持)
基于宽表预构建的方式在 ES 上有较好的索引、查询性能。
- 缺点：
反查时数据可能没有准备好，导致数据缺失（这里具体的影响取决于数据同步工具本身设计，可以引入时间窗口配合超时等待，也可以没有数据时直接同步到对端）
需要数据同步工具在数据反查、数据加工方面进行支持
- 总结
对于构建宽表涉及数据加工的场景，该方式比较适合。
由于该方式的回查机制、预构建前数据加工的能力支持，能力上是“多表订阅合并预构建宽表数据”这种方式的超集。如果有比较好的数据同步工具支持，这种方式是比较推荐的。


#####工具选型
阿里云： 
DTS，DataWorks
自建：
logstash-input-jdbc  同步全量数据，接受秒级延迟，批量查询数据然后进行同步的场景。
Canal 对数据同步的实时性要求较高的场景。
elasticsearch-jdbc  能实现mysql数据全量和增量的数据同步.不能实现同步删除操作
go-mysql-elasticsearch  能实现mysql数据增加,删除,修改操作的实时数据同步






