# Profiling Aggregations

## 聚合部分/aggregations Section

聚合部分(aggregations)包含在一个特定的分块执行生成的聚合树的详细时序。这种聚合树的整体结构类似于原来的Elasticsearch的请求。让我们分析以下示例聚合请求：

```bash
GET /house-prices/_search
{
  "profile": true,
  "size": 0,
  "aggs": {
    "property_type": {
      "terms": {
        "field": "propertyType"
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```
这导致下列的聚合探查(aggregation profile)输出

```js
"aggregations": [
  {
    "type": "org.elasticsearch.search.aggregations.bucket.terms.GlobalOrdinalsStringTermsAggregator",
    "description": "property_type",
    "time": "4280.456978ms",
    "time_in_nanos": "4280456978",
    "breakdown": {
      "reduce": 0,
      "reduce_count": 0,
      "build_aggregation": 49765,
      "build_aggregation_count": 300,
      "initialise": 52785,
      "initialize_count": 300,
      "collect": 3155490036,
      "collect_count": 1800
    },
    "children": [
      {
        "type": "org.elasticsearch.search.aggregations.metrics.avg.AvgAggregator",
        "description": "avg_price",
        "time": "1124.864392ms",
        "time_in_nanos": "1124864392",
        "breakdown": {
          "reduce": 0,
          "reduce_count": 0,
          "build_aggregation": 1394,
          "build_aggregation_count": 150,
          "initialise": 2883,
          "initialize_count": 150,
          "collect": 1124860115,
          "collect_count": 900
        }
      }
    ]
  }
]
```

根据探查(Profile)的结构，我们可以看到聚合property_type内部由GlobalOrdinalsStringTermsAggregator类表示及子聚合avg_price内部由AvgAggregato类表示。

时间字段(time)显示，整个聚合要花费4秒执行。记录的时间包括所有孩子节点。

崩溃字段(breakdown)给出时间如何花费的详细数据，我们一眼可以看到它。最后，孩子(children)的数组列表列出了所有可能出现的子聚合。因为我们有一个属于property_type的子聚合avg_price，我们可以看到，它被列为聚合property_type的孩子。两个聚合的输出有相同的信息（类型、时间、故障等）。孩子(children)可以嵌套自己的孩子(children)。

## 定时故障/Timing Breakdown

故障组件(breakdown )列出了关于低级别Lucene执行的详细时序统计信息：

```bash
"breakdown": {
  "reduce": 0,
  "reduce_count": 0,
  "build_aggregation": 49765,
  "build_aggregation_count": 300,
  "initialise": 52785,
  "initialize_count": 300,
  "collect": 3155490036,
  "collect_count": 1800
}
```
时间信息用网络挂钟的纳秒列出来，且不规范化。所有关于时间的警告均适用于这里。崩溃字段 (breakdown)的意图是让你感觉到（A）Elasticsearch的运转实际上耗费时间的，（B）各部件耗费时间的差异是非常大的。像总的时间一样，故障包含所有孩子的时间。

该属性的含义如下：

## 所有的参数：/All parameters:

名称   | 描述
-----|--------
`initialise` | 在开始收集文件前，创建和初始化聚合所需要的时间。
`collect` | 这表示聚合部分的聚合阶段累积所花时间。这是匹配的文档传递到聚合的地方，且聚合状态是基于文档中所包含信息的进行更新的。
`build_aggregation` | 这代表了聚合部分在文档集合完毕后准备回传到减少节点产生的创造碎片级别的时间。
`reduce` | 这不是目前使用的，而且总是会报告0。目前聚合探查(aggregation profiling)只计时部分聚合执行的碎片水平。 减少阶段的计时将稍后添加。
`*_count` | 记录的特定方法的调用次数。例如，`"collect_count": 2`，意味着`collect()`方法被两个不同的文件调用。