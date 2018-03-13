# Count API

Count API 允许轻松执行查询并获取该查询的匹配数。 它可以跨一个或多个索引并跨越一个或多个类型执行。 可以使用简单的查询字符串作为参数或使用在请求正文中定义的[Query DSL](../Query_DSL.md)来提供查询。 这里是一个例子：

```bash
PUT /twitter/tweet/1?refresh
{
    "user": "kimchy"
}
 
GET /twitter/tweet/_count?q=user:kimchy
 
GET /twitter/tweet/_count
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

> 注意
>
> 在正文中发送的查询必须嵌套在查询键中，与[Search API](../Search_APIs/Search.md)相同

上面的两个例子都做同样的事情，这是从某个用户的 twitter 索引计数 tweets 的数量。 其结果是：

```bash
{
    "count" : 1,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

查询是可选的，如果没有提供，它将使用`match_all`来计算所有的文档。

## Multi index, Multi type （多索引，多类型）

count API 可以应用于[多个索引中的多个类型](../Search_APIs/Search.md#search-multi-index-type)。

## Request Parameters （请求参数）

当使用查询参数`q`执行计数时，传递的查询是使用Lucene查询解析器的查询字符串。 还有其他可以传递的参数：

名称     | 描述
------|-------
`df` | 在查询中未定义字段前缀时使用的默认字段。
`analyzer` | 分析查询字符串时使用的分析器名称。
`default_operator` | 要使用的默认运算符，可以是`AND`或`OR`。 默认为`OR`。
`lenient` | 如果设置为`true`将导致基于格式的失败（例如向数字字段提供文本）被忽略。 默认为`false`。
`lowercase_expanded_terms` | 术语是否自动小写，默认为`true`。
`analyze_wildcard` | 是否分析通配符和前缀查询。 默认为`false`。
`terminate_after` | 每个分片的最大计数，到达时，查询执行将提前终止。 如果设置，响应将具有布尔字段`terminated_early`以指示查询执行是否实际已终止。 默认为无`terminate_after`。

## Request Body | （请求主体）

计数可以使用其身体内的[Query DSL](../Query_DSL.md)来表达应该执行的查询。 主体内容也可以作为名为`source`的 REST 参数传递。

HTTP GET 和 HTTP POST 都可以用于以主体执行计数。 由于并不是所有的客户端都支持带主体的 GET，因此也允许 POST。

## Distributed （分布式）

计数操作在所有分片上广播。 每个 shard id group 选择一个副本并对其执行。 这意味着副本增加了计数的可伸缩性。

##Routing （路由）

可以指定路由值（路由值的逗号分隔列表），以控制将对哪些分片执行计数请求。