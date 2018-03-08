# Preference

`perference`可以控制执行搜索请求的分片副本。默认情况下，操作在分片副本之间是随机化的。

`preference`是一个查询字符串参数，可以设置为：

值选项          | 说明
----------|---------------
`_primary`      | 操作将继续并只在主分片上执行。
`_primary_first` | 操作将在主分片上执行，如果不可用（故障转移），将在其他分片上执行。
`_replica`       | 该操作将只在副本分片上执行。
`_replica_first`| 操作将移动并仅在副本分片上执行，如果不可用（故障转移），则将在其他分片上执行。
`_local`          | 如果可能，操作将优选在本地分配的分片上执行。
`_prefer_nodes:abc,xyz` | 在适用的情况下，在具有提供的节点标识（本例中为`abc`或`xyz`）的节点上优先执行。
`_shards:2,3`  | 将操作限制为指定的分片。（在这种情况下为`2`和`3`）。此首选项可以与其他首选项组合，但必须首先显示：`_shards:2,3|_primary`
`_only_nodes` | 将操作限制在节点说明中[指定的节点](../../Cluster_APIs.md)
Custom (string) value| 自定义值将用于保证相同的自定义值使用相同的分片。当在不同的刷新状态中匹配不同的分片时，这可以帮助“跳跃值”。示例值可以是Web的`session id`或`user name`。

例如，使用用户的`session id`来确保用户的结果的一致排序：

```bash
GET /_search?preference=xyzabc123
{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    }
}
```
