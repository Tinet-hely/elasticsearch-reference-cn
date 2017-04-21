# Delete API（删除接口）

delete API允许基于指定的ID来从索引库中删除一个JSON文件。下面演示了从一个叫`twitter`的索引库的`tweet` type下删除文档，`id`是`1`:

```bash
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1'
```

上述删除操作的结果是：

```js
{
    "_shards" : {
        "total" : 10,
        "failed" : 0,
        "successful" : 10
    },
    "found" : true,
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "1",
    "_version" : 2,
    "result": "deleted"
}
```

## 版本

索引的每个文档都被标记了版本。当删除文档时， 可以通过指定`version`来确保我们试图删除一个实际上已被删除的文档时，它在此期间并没有改变。在文档中执行的每个写入操作，包括删除，都会使其版本递增。

## 路由

在创建索引文档时如果使用了控制路由的能力，为了删除文档，也应当提供路由值。例如：

```bash
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?routing=kimchy'
```

以上将删除ID为1的tweet，但会根据用户路由。请注意，如果删除路由值不正确，会导致文档无法删除。

当映射的`_routing`被设定为`required`且没有指定的路由值时，删除API将抛出`RoutingMissingException`并拒绝该请求。

## Parent

`parent`参数可以被设置，这将基本上与设定路由参数是相同的。

请注意，删除父文档不会自动删除其子文档。根据给定的父文档ID删除所有子文件的一种方法是，通过在创建文档索引时自动生成的`_parent`字段来使用[根据查询条件删除API](./Delete_By_Query_API.md)进行删除，它的格式是`parent_type#parent_id`。

当删除子文档，必须指定其父ID，否则该删除请求将被拒绝和抛出一个`RoutingMissingException`异常。

## 自动创建索引

如果索引库之前没有创建，删除操作将自动创建一个索引库（参见[创建索引API](../Indices_APIs/Create_Index.md)来手动创建索引），并且如果没有创建类型时，会根据指定的类型名与动态映射类型来自动创建类型（参见[put mapping](../Indices_APIs/Put_Mapping.md)来手动创建类型映射）。

## 分布式

删除操作被散列到一个特定的分片id。然后它被重定向到该ID组内的主分片，和副本分片（如果需要的话）。

## 等待活动分片

当进行的删除请求，你可以设置`wait_for_active_shards`参数来要求必须最少达到几个可用的分片才能开始处理删除请求。进一步的细节和使用示例见[这里](./Index_API.md#index-wait-for-active-shards)。

## 刷新

用来控制本次的变化能够被搜索可见。参见：[refresh](./refresh.md)。

## 超时

在执行删除操作时，分配给执行删除操作的主分片可能无法使用。有些方面的原因可能是主分片正在从仓库恢复或进行搬迁。默认情况下，删除操作在返回失败与错误之前将等待1分钟让主分片成为可用的。该`timeout`参数可用于明确指定等待多长时间。这里是将其设置为5分钟的一个示例：

```bash
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?timeout=5m'
```