# Profiling Queries

> 注意
> 
> Profile API 提供的详细信息直接暴露了Lucene的类名和概念，这意味着对结果的完整解释需要Lucene相当高级的知识。本页试图对Lucene如何执行查询提供速成教程，以便您可以成功地使用Profile API诊断和调试查询，但它只是一个概述。如需完整的理解，请参考Lucene对应位置的文档和代码。
>
> 也就是说，处理一个缓慢的查询往往不需要完整理解Lucene。例如，我们普遍知道某个特定的查询组件很缓慢，但不一定理解为什么该查询的advance前阶段是诱因。

## 查询部分/query Section

查询部分(query)包含由Lucene在一个特定的分块执行生成的查询树的详细时序。这个查询树的整体结构类似于原来的Elasticsearch查询，但可能会略有不同（偶尔会差异很大）。它也将使用类似但不总是相同的命名。使用我们以前的匹配查询(match)示例，让我们分析查询部分(query)：

```bash
"query": [
    {
       "type": "BooleanQuery",
       "description": "message:message message:number",
       "time": "1.873811000ms",
       "time_in_nanos": "1873811",
       "breakdown": {...},               # ①
       "children": [
          {
             "type": "TermQuery",
             "description": "message:message",
             "time": "0.3919430000ms",
             "time_in_nanos": "391943",
             "breakdown": {...}
          },
          {
             "type": "TermQuery",
             "description": "message:number",
             "time": "0.2106820000ms",
             "time_in_nanos": "210682",
             "breakdown": {...}
          }
       ]
    }
]
```

① 为简单起见，这里省略故障时间。
__________________

基于探查(profile)的结构，我们可以看到，我们的匹配查询(match)被Lucene重写为包含两个条款(均有术语查询(TermQuery))的布尔查询(BooleanQuery)。类型字段(type)显示Lucene类的名称，并经常与Elasticsearch中对应的名字相同。这个描述字段(description)显示Lucene查询的解析文本,并可用于帮助区分查询的各个部分。（如：message:search and message:test 都是术语查询(TermQuery)，否则会出现相同的两个。）

时间字段(time)表明该查询了执行整个布尔查询(BooleanQuery)花费1.8ms，此记录时间包含了所有孩子节点。

time_in nanos字段显示一个精确的、机器可读格式的时间信息(以纳秒为单位)。

崩溃字段(breakdown)给出时间如何花费的详细数据，我们一眼可以看到它。最后，孩子(children)数组列出了所有可能出现的子查询。因为我们搜索了两个值（“search test”），布尔查询(BooleanQuery)有两个孩子术语查询(TermQueries)。它们有相同的信息（类型、时间、故障等）。孩子(children)可以嵌套自己的孩子(children)。

> 注意
>
> 时间字段(time)仅用于人类消费。如果你需要精确的定时值请使用`time_in nanos`字段。目前，默认打印时间字段(time)，但这将在下一个主要版本 (6.0.0)的发生变化，将默认打印`time_in_nanos`字段。

## 定时故障/Timing Breakdown

崩溃组件(breakdown)列出底层Lucene执行的详细时序统计：

```js
"breakdown": {
   "score": 51306,
   "score_count": 4,
   "build_scorer": 2935582,
   "build_scorer_count": 1,
   "match": 0,
   "match_count": 0,
   "create_weight": 919297,
   "create_weight_count": 1,
   "next_doc": 53876,
   "next_doc_count": 5,
   "advance": 0,
   "advance_count": 0
}
```

时间信息用网络挂钟的纳秒列出来，且不规范化。所有关于时间的警告均适用于这里。 Breakdown的意图是让你感觉到（A）Lucene的运转实际上耗费时间的，（B）各部件耗费时间的差异是非常大的。像所有的时间一样，breakdown包含所有孩子的时间。

统计数据的含义如下：

### 所有的参数：/All parameters:

参数        |   描述
-------|--------
`create_weight` | Lucene的查询必须能够在复杂的IndexSearchers重用（它被看做是针对特定的Lucene索引执行搜索的引擎。）。这使得Lucene处于一个棘手的境地，因为许多查询(Query)需要积累与它正在使用的索引相关联的临时的状态/统计信息，但查询(Query)合同授权要求它必须不可变的。为了解决这个问题，Lucene要求每个查询生成一个权重对象(weight object)作为临时的上下文对象为这个特定的元组(IndexSearcher，Query)保持状态信息。weight的度量表明这个过程所需要的时间长短。
`build_scorer` | 此参数显示建立查询的记分器(Scorer)需要多长的时间。记分器(Scorer)是一种遍历所有匹配文档为每个文档生成得分的机制（例如，“foo”与文档的匹配程度是怎样的？）。注意，这记录了生成记分器(Scorer)对象而不是对文档进行评分所需的时间。不同查询初始化记分器(Scorer)有快有慢，取决于优化、复杂性等。这也可能说明计时与缓存(caching)是否被启用或缓存是否适用于查询(query)相关。
`next_doc` |  Lucene的方法next_doc返回下一个匹配查询的文档ID。此统计数据显示确定哪个文档是下一个匹配需要的时间，这是根据查询的性质而变化很大的过程。next_doc是一种特殊形式的advance()，它使得Lucene的许多查询更便捷。这相当于函数advance(docId() + 1)。
`advance` | advance是next_doc的低版本：它的目的是找到下一个匹配的DOC，但需要调用查询执行额外任务，如识别和移动过去的跳跃。然而，不是所有的查询都可以使用next_doc，所以advance的目的是服务于那些查询。联合查询(Conjunctions)（如，布尔查询中有must）是advance的典型消费者。
`matches	` | 一些查询，如短语查询，使用“两阶段”过程匹配文档。首先，“近似”匹配文档，如果文档大约匹配，它将用更严格的（和昂贵的）的方法进行第二次检查。第二阶段验证是匹配的统计测量。例如，短语查询首先通过确保所有术语都存在于文档中来大约检查文档。如果所有的术语都存在，那么它执行第二阶段验证，以确保条款按次序形成短语，相比检查条款是否存在这是更昂贵的。由于这两过程仅被少数查询使用，统计度量结果往往是零。
`score` | 这记录了一个特定的文件通过评分器(Scorer)评分所需的时间。
`*_count	` | 纪录调用特定方法的数量。例如， ”next_doc_count”：2，意味着nextDoc()在两个不同的文档中被调用。这可以通过比较不同查询组件之间的计数来帮助判断如何选择查询。

## 收集部分/collectors Section
响应的收集器(Collectors)部分显示高级执行细节。Lucene通过定义一个“收集器(Collector)”来工作，它负责协调匹配文档的遍历、得分和集合。收集器(Collectors)也有单个查询如何记录聚合结果、执行无作用域的“global”查询、执行post-query过滤，等功能。

看前面的例子：

```js
"collector": [
   {
      "name": "CancellableCollector",
      "reason": "search_cancelled",
      "time": "0.3043110000ms",
      "time_in_nanos": "304311",
      "children": [
        {
          "name": "SimpleTopScoreDocCollector",
          "reason": "search_top_hits",
          "time": "0.03227300000ms",
          "time_in_nanos": "32273"
        }
      ]
   }
]
```

我们看到一个收集器(Collector)由SimpleTopScoreDocCollector包装成 CancellableCollector。SimpleTopScoreDocCollector是Elasticsearch使用的默认的“评分和排序”的收集器(Collector)。原因字段(reason)试图对类名进行简单的英文描述。时间字段(time)与查询树中的时间字段(time)相似：一个包括所有孩子节点的网络挂钟时间。同样的是，孩子(children)列出所有子收集器(Collector)。包装SimpleTopScoreDocCollector的CancellableCollector，被Elasticsearch用于检测当前搜索是否被取消，一旦发生取消搜索的行为则停止收集文件。

应该指出的是，Collector times与Query times相互独立。他们独立计算、合并和规范化！由于Lucene的执行的性质，它不可能把收集器(Collectors)的时间“合并“”到查询部分(Query)，所以他们在不同的部分显示出来。

作为参考，各种收集器Collectors的原因是：

名称    |  描述
-----|-----------
`search_sorted` | 整理和分类文件的收集器(collector)。这是最常见的收集器，会最简单的搜索中出现。
`search_count` | 一个仅计算查询匹配的文档数的收集器(collector)，但不能获取源代码。只有当参数size被指定为0，这个收集器才会出现。
`search_terminate_after_count` | 一个当N个匹配文档被发现就终止搜索的收集器(collector)。只有当terminate_after_count query参数被指定，这个收集器才会出现。
`search_min_score` | 一个只返回评分大于 N的匹配文档的收集器(collector)。只有当顶层参数min_score被指定，这个收集器才会出现。
`search_multi` | 包裹其他几个收集器的收集器(collector)。只有当组合搜索(combinations of search)，聚合(aggregations)，全局聚合(global aggs)和post_filters结合在一个搜索里，这个收集器才会出现。
`search_timeout` | 一个在特定时间中断执行的收集器(collector)。只有当顶层参数timeout被指定，这个收集器才会出现。
`aggregation` | 一个Elasticsearch在查询范围使用聚合的收集器(collector)。一个为所有聚合采集文件的聚合采集器，所以你会看到一个聚合名称的列表。
`global_aggregation` | 一个对全局查询(global query scope)而不是指定查询(specified query)执行聚合(aggregation)的收集器(Collector)。由于全局范围(global query)不同于执行普通查询(query)，它必须执行它自己的match_all查询（这会被添加到查询部分(Query)）来收集整个数据集。


## 重写部分/rewrite Section

Lucene中的所有查询都经过“重写”过程。一个查询（及其子查询）可以重写一次或多次，这过程继续进行，直到查询停止更改。这个过程让Lucene进行优化，如去除多余的条款，一个更有效的执行路径替换一个查询。例如Boolean → Boolean → TermQuery 可以改写为术语查询(TermQuery)，因为在这种情况下所有的布尔值都是多余的。

重写的过程是复杂的，难以显示，因为查询可以大幅改变。总改写时间不显示中间结果，只是显示为一个值（以纳秒为单位）。此值是累加的，包含所有被重写查询的总时间。

## 更复杂的例子/A more complex example

为了演示稍微复杂的查询和相关的结果，我们可以探查(profile)以下查询：

```bash
GET /test/_search
{
  "profile": true,
  "query": {
    "term": {
      "message": {
        "value": "search"
      }
    }
  },
  "aggs": {
    "non_global_term": {
      "terms": {
        "field": "agg"
      },
      "aggs": {
        "second_term": {
          "terms": {
            "field": "sub_agg"
          }
        }
      }
    },
    "another_agg": {
      "cardinality": {
        "field": "aggB"
      }
    },
    "global_agg": {
      "global": {},
      "aggs": {
        "my_agg2": {
          "terms": {
            "field": "globalAgg"
          }
        }
      }
    }
  },
  "post_filter": {
    "term": {
      "my_field": "foo"
    }
  }
}
```

这个例子有：

- 一个查询(query)
- 一个局部聚合(scoped aggregation)
- 一个全局聚合(global aggregation)
- 一个后过滤(post_filter)

响应：

```js
{
   "profile": {
         "shards": [
            {
               "id": "[P6-vulHtQRWuD4YnubWb7A][test][0]",
               "searches": [
                  {
                     "query": [
                        {
                           "type": "TermQuery",
                           "description": "my_field:foo",
                           "time": "0.4094560000ms",
                           "time_in_nanos": "409456",
                           "breakdown": {
                              "score": 0,
                              "score_count": 1,
                              "next_doc": 0,
                              "next_doc_count": 2,
                              "match": 0,
                              "match_count": 0,
                              "create_weight": 31584,
                              "create_weight_count": 1,
                              "build_scorer": 377872,
                              "build_scorer_count": 1,
                              "advance": 0,
                              "advance_count": 0
                           }
                        },
                        {
                           "type": "TermQuery",
                           "description": "message:search",
                           "time": "0.3037020000ms",
                           "time_in_nanos": "303702",
                           "breakdown": {
                              "score": 0,
                              "score_count": 1,
                              "next_doc": 5936,
                              "next_doc_count": 2,
                              "match": 0,
                              "match_count": 0,
                              "create_weight": 185215,
                              "create_weight_count": 1,
                              "build_scorer": 112551,
                              "build_scorer_count": 1,
                              "advance": 0,
                              "advance_count": 0
                           }
                        }
                     ],
                     "rewrite_time": 7208,
                     "collector": [
                        {
                           "name": "MultiCollector",
                           "reason": "search_multi",
                           "time": "1.378943000ms",
                           "time_in_nanos": "1378943",
                           "children": [
                              {
                                 "name": "FilteredCollector",
                                 "reason": "search_post_filter",
                                 "time": "0.4036590000ms",
                                 "time_in_nanos": "403659",
                                 "children": [
                                    {
                                       "name": "SimpleTopScoreDocCollector",
                                       "reason": "search_top_hits",
                                       "time": "0.006391000000ms",
                                       "time_in_nanos": "6391"
                                    }
                                 ]
                              },
                              {
                                 "name": "BucketCollector: [[non_global_term, another_agg]]",
                                 "reason": "aggregation",
                                 "time": "0.9546020000ms",
                                 "time_in_nanos": "954602"
                              }
                           ]
                        }
                     ]
                  },
                  {
                     "query": [
                        {
                           "type": "MatchAllDocsQuery",
                           "description": "*:*",
                           "time": "0.04829300000ms",
                           "time_in_nanos": "48293",
                           "breakdown": {
                              "score": 0,
                              "score_count": 1,
                              "next_doc": 3672,
                              "next_doc_count": 2,
                              "match": 0,
                              "match_count": 0,
                              "create_weight": 6311,
                              "create_weight_count": 1,
                              "build_scorer": 38310,
                              "build_scorer_count": 1,
                              "advance": 0,
                              "advance_count": 0
                           }
                        }
                     ],
                     "rewrite_time": 1067,
                     "collector": [
                        {
                           "name": "GlobalAggregator: [global_agg]",
                           "reason": "aggregation_global",
                           "time": "0.1226310000ms",
                           "time_in_nanos": "122631"
                        }
                     ]
                  }
               ]
            }
         ]
      }
}
```

正如你所看到的，输出明显比前面冗长。查询的所有主要部分都表示：

1. 第一个TermQuery (message:search) 代表主术语查询。

1. 第二个TermQuery (my_field:foo) 代表后过滤(post_filter)查询。

1. 有一个MatchAllDocsQuery （*：*）查询，作为执行第二个不同的搜索。这不是由用户指定的查询的一部分，而是由全局聚合(global aggregation)为提供全局查询范围而自动生成的。

收集树是相当简单的，显示了一个MultiCollector如何包裹FilteredCollector去执行post_filter（反过来，包裹正常的评分SimpleCollector），和bucketcollector运行所有作用域的聚合。 In the MatchAll search, there is a single GlobalAggregator to run the global aggregation.在MatchAll搜索中，有一个全局聚合器(GlobalAggregator)运行全局的聚合。

## 了解MultiTermQuery的输出/Understanding MultiTermQuery output

这里需要对MultiTermQuery类查询做一个特别注释。这包括通配符(wildcards)，正则表达式(regex)和模糊(fuzzy)查询。这些查询发出非常冗长的响应，并且不过度结构化。

从本质上讲，这些查询(query)在每一个段的基础上改写自己。如果你想象中的通配符查询为"b*"，在技术上它可以匹配任何以字母“b”开头的标记。无法枚举所有可能的组合，所以Lucene重写查询中被评估的段落。例如，某段中可能包含标记 [bar, baz]，所以查询query重写到布尔查询(BooleanQuery)中包含了"bar"和"baz"。另一段可能只有标记 [bakery]，所以查询query重写成只包含"bakery"的术语查询(TermQuery)。

由于这种每段重写的动态，干净的树结构变得扭曲，并且不再有清晰的世系去显示一个查询如何被重写rewriter成下一个。目前，我们所能做的就是道歉，如果它太混乱，建议您检查该查询的孩子节点的崩溃的细节。幸运的是，所有的时间统计都是正确的，只是不在响应(response)的物理布局中，因此只需分析顶层的MultiTermQuery，如果你发现的细节很难解释请忽视的它的孩子节点。

希望在未来的迭代它会变成固定，但它是一个很难解决的、正在改善中的棘手问题 :)