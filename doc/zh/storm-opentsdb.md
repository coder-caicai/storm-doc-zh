---
title: Storm OpenTSDB 集成
layout: documentation
documentation: true
---

# Storm OpenTSDB Bolt 和 TridentState

OpenTSDB 为时间序列数据提供了可扩展且高可用性的存储.
它由 Time Series Daemon (TSD) servers 以及命令行工具组成.
每个 TSD 连接到配置的 HBase 集群以 push/query（推送/查询） 数据.

时间序列数据点包括:
 - a metric name.
 - a UNIX timestamp (seconds or milliseconds since Epoch).
 - a value (64 bit integer or single-precision floating point value).
 - a set of tags (key-value pairs) that describe the time series the point belongs to.

Storm bolt 和 trident state 从一个基于给定的 `TupleMetricPointMapper` 的 tuple 中创建了上面的时间序列数据.

该模块提供了 core Storm 和 Trident bolt 实现, 用户将数据写入 OpenTSDB.

时间序列数据点被写入时有 at-least-once（至少一次）的语义保证, 并且重复的数据点应该像 OpenTSDB 的 [这里](http://opentsdb.net/docs/build/html/user_guide/writing.html#duplicate-data-points) 提及的一样来处理.

## 示例

### Core Bolt
下面的示例描述了 `org.apache.storm.opentsdb.bolt.OpenTsdbBolt` cord bolt 的使用方法

```java

        OpenTsdbClient.Builder builder =  OpenTsdbClient.newBuilder(openTsdbUrl).sync(30_000).returnDetails();
        final OpenTsdbBolt openTsdbBolt = new OpenTsdbBolt(builder, Collections.singletonList(TupleOpenTsdbDatapointMapper.DEFAULT_MAPPER));
        openTsdbBolt.withBatchSize(10).withFlushInterval(2000);
        topologyBuilder.setBolt("opentsdb", openTsdbBolt).shuffleGrouping("metric-gen");
        
```


### Trident State

```java

        final OpenTsdbStateFactory openTsdbStateFactory =
                new OpenTsdbStateFactory(OpenTsdbClient.newBuilder(tsdbUrl),
                        Collections.singletonList(TupleOpenTsdbDatapointMapper.DEFAULT_MAPPER));
        TridentTopology tridentTopology = new TridentTopology();
        
        final Stream stream = tridentTopology.newStream("metric-tsdb-stream", new MetricGenSpout());
        
        stream.peek(new Consumer() {
            @Override
            public void accept(TridentTuple input) {
                LOG.info("########### Received tuple: [{}]", input);
            }
        }).partitionPersist(openTsdbStateFactory, MetricGenSpout.DEFAULT_METRIC_FIELDS, new OpenTsdbStateUpdater());
        
```
