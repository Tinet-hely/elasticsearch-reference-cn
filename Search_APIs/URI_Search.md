# URI Search

可以通过提供请求参数来纯粹使用 URI 来执行搜索请求。 在使用此模式执行搜索时，并非所有搜索选项都会公开，但它可以方便快速的进行“curl 测试”。

这里给出一个例子：

```js
GET twitter/tweet/_search?q=user:kimchy
```

并给出一个示例响应：

```js
{
    "timed_out": false,
    "took": 62,
    "_shards":{
        "total" : 1,
        "successful" : 1,
        "failed" : 0
    },
    "hits":{
        "total" : 1,
        "max_score": 1.3862944,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "0",
                "_score": 1.3862944,
                "_source" : {
                    "user" : "kimchy",
                    "date" : "2009-11-15T14:12:12",
                    "message" : "trying out Elasticsearch",
                    "likes": 0
                }
            }
        ]
    }
}
```

## 参数

URI 中允许使用的参数有:

参数名              | 描述
-------------------|--------------------
`q`                | 查询字符串(映射到`query_string`查询，有关更多详细信息，请参阅查询字符串查询）
`df`               | 在查询中未定义字段前缀时使用的默认字段。
`analyzer`         | 分析查询字符串时使用的分析器名称。
`lowercase_expanded_terms` | 应将条款自动缩小或不缩小。默认为`true`。
`analyze_wildcard` | 应该分析通配符和前缀查询还是不分析。默认为`false`。
`default_operator` | 要使用的默认运算符，可以是`AND`或`OR`。默认为`OR`。
`lenient`          | 如果设置为`true`将导致基于格式的失败（例如向数字字段提供文本）被忽略。默认为`false`。
`explain`          | 对于每个命中，包含对如何计算命中的计分的解释。
`_source`          |设置为`false`以禁用检索`_source`字段。您还可以使用 `_source_include`＆`_source_exclude`检索文档的一部分（有关更多详细信息，请参阅请求主体文档）。
`stored_fields`    | 为每次命中返回文档的选择性存储字段，逗号分隔。未指定任何值将不会返回任何字段。
`sort`             | 排序执行。可以是`fieldName`或`fieldName:asc`/`fieldName:desc`的形式。 `fieldName`可以是文档中的实际字段，也可以是指示基于分数排序的特殊`_score`名称。可以有几个`sort`参数（顺序很重要）。
`track_scores`     | 排序时，设置为`true`以便仍然跟踪分数并将其作为每次匹配的一部分返回。
`timeout`          | 搜索超时，将搜索请求限制为在指定的时间值内执行并且保留与到期时累积的点击数。默认为无超时。
`terminate_after`  | 要为每个分片收集的文档的最大数量，到达时，查询执行将提前终止。如果设置，响应将有布尔型字段`terminated_early`以指示查询执行是否实际已提前终止。默认为无`terminate_after`。
`from`             | 从命中的索引开始返回。默认值为`0`。
`size`             | 要返回的匹配数。默认值为`10`。
`search_type`      | 要执行的搜索操作的类型。可以是`dfs_query_then_fetch`或`query_then_fetch`。默认为`query_then_fetch`。有关可以执行的不同类型搜索的更多详细信息，请参阅搜索类型。