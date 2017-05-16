# Reindex API

> 重要
>
> Reindex不会尝试设置目标索引。它不会复制源索引的设置。您应该在运行`_reindex`操作之前设置目标索引，包括设置映射，分片数，副本等。

`_reindex`的最基本形式只是将文档从一个索引复制到另一个索引。下面将文档从`twitter`索引复制到`new_twitter`索引中：

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
COPY
```

这将会返回类似以下的东西：

```js
{
  "took" : 147,
  "timed_out": false,
  "created": 120,
  "updated": 0,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "total": 120,
  "failures" : [ ]
}
```

就像[_update_by_query](./Update_By_Query_API.md)一样，`_reindex`获取源索引的快照，但其目标必须是**不同**的索引，因此版本冲突是不可能的。 `dest`元素可以像索引API一样进行配置，以控制乐观并发控制。只需将`version_type`（如上所述）或将其设置为`internal`将导致Elasticsearch盲目将文档转储到目标中，覆盖具有相同类型和ID的任何内容：

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}
```

将`version_type`设置为`external`将导致Elasticsearch从源文件中保留版本，创建缺失的所有文档，并更新在目标索引中比源索引中版本更老的所有文档：

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  }
}
```

设置`op_type`为`create`将导致`_reindex`仅在目标索引中创建缺少的文档。所有存在的文档将导致版本冲突：

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```

默认情况下，版本冲突将中止`_reindex`进程，但您可以通过请求体设置`"conflict":"proceed"`来在冲突时进行计数：

```js
POST _reindex
{
  "conflicts": "proceed",
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```

您可以通过向`source`添加`type`或添加`query`来限制文档。下面会将`kimchy`发布的`tweet`复制到`new_twitter`中：

```js
POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "tweet",
    "query": {
      "term": {
        "user": "kimchy"
      }
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

`source`中的`index`和`type`都可以是一个列表，允许您在一个请求中从大量的来源进行复制。下面将从`twitter`和`blog`索引中的`tweet`和`post`类型中复制文档。它也包含`twitter`索引中`post`类型以及`blog`索引中的`tweet`类型。如果你想更具体，你将需要使用`query`。它也没有努力处理ID冲突。目标索引将保持有效，但由于迭代顺序定义不正确，预测哪个文档可以保存下来是不容易的。

```js
POST _reindex
{
  "source": {
    "index": ["twitter", "blog"],
    "type": ["tweet", "post"]
  },
  "dest": {
    "index": "all_together"
  }
}
```

还可以通过设置大小限制处理的文档的数量。下面只会将单个文档从`twitter`复制到`new_twitter`：

```js
POST _reindex
{
  "size": 1,
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

如果你想要从`twitter`索引获得一个特定的文档集合你需要排序。排序使滚动效率更低，但在某些情况下它是值得的。如果可能，更喜欢更多的选择性查询`size`和`sort`。这将从`twitter复`制`10000`个文档到`new_twitter`：

```js
POST _reindex
{
  "size": 10000,
  "source": {
    "index": "twitter",
    "sort": { "date": "desc" }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

`source`部分支持[搜索请求](../Search_APIs/Request_Body_Search.md)中支持的所有元素。例如，只使用原始文档的一部分字段，使用源过滤如下所示：

```js
POST _reindex
{
  "source": {
    "index": "twitter",
    "_source": ["user", "tweet"]
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

像`update_by_query`一样，`_reindex`支持修改文档的脚本。与`_update_by_query`不同，脚本允许修改文档的元数据。此示例修改了源文档的版本：

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  },
  "script": {
    "inline": "if (ctx._source.foo == 'bar') {ctx._version++; ctx._source.remove('foo')}",
    "lang": "painless"
  }
}
```

就像在`_update_by_query`中一样，您可以设置`ctx.op`来更改在目标索引上执行的操作：

`noop`

  如果您的脚本决定不必进行任何更改，请设置 `ctx.op ="noop"` 。这将导致`_update_by_query` 从其更新中忽略该文档。这个没有操作将被报告在[响应体](./Update_By_Query_API.md#response-body)的 `noop` 计数器上。

`delete`

  如果您的脚本决定必须删除该文档，请设置`ctx.op="delete"`。删除将在[响应体](#response-body)的 `deleted` 计数器中报告。

将`ctx.op`设置为其他任何内容都是错误。在`ctx`中设置任何其他字段是一个错误。

想想可能性！只要小心点，有很大的力量...你可以改变：

- `_id`
- `_type`
- `_index`
- `_version`
- `_routing`
- `_parent`

将`_version`设置为`null`或从`ctx`映射清除就像在索引请求中不发送版本一样。这将导致目标索引中的文档被覆盖，无论目标版本或`_reindex`请求中使用的版本类型如何。

默认情况下，如果`_reindex`看到具有路由的文档，则路由将被保留，除非脚本被更改。您可以根据`dest`请求设置`routing`来更改：

`keep`

    将批量请求的每个匹配项的路由设置为匹配上的路由。默认值。

`discard`

    将批量请求的每个匹配项的路由设置为null。

`=<某些文本>`

     将批量请求的每个匹配项的路由设置为`=`之后的文本。

例如，您可以使用以下请求将`source`索引的所有公司名称为`cat`的文档复制到路由设置为`cat`的`dest`索引。

```js
POST _reindex
{
  "source": {
    "index": "source",
    "query": {
      "match": {
        "company": "cat"
      }
    }
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}
```

默认情况下，`_reindex`批量滚动处理大小为`1000`.您可以在`source`元素中指定`size`字段来更改批量处理大小：

```js
POST _reindex
{
  "source": {
    "index": "source",
    "size": 100
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}
```

Reindex也可以使用[Ingest Node]功能来指定`pipeline`, 就像这样：

```js
POST _reindex
{
  "source": {
    "index": "source"
  },
  "dest": {
    "index": "dest",
    "pipeline": "some_ingest_pipeline"
  }
}
```

## <span id="#reindex-from-remote">从远程重建索引</span>

Reindex支持从远程Elasticsearch群集重建索引：

```js
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

`host`参数必须包含`scheme`，`host`和`port`（例如 `https：// otherhost:9200`）。用户名和密码参数是可选的，当它们存在时，索引将使用基本认证连接到远程Elasticsearch节点。使用基本认证时请务必使用`https`，密码将以纯文本格式发送。

必须在`elasticsearch.yaml`中使用`reindex.remote.whitelist`属性将远程主机明确列入白名单。它可以设置为允许的远程`host`和`port`组合的逗号分隔列表（例如`otherhost:9200,another:9200,127.0.10.*:9200,localhost:*`）。白名单忽略了`scheme` ——仅使用主机和端口。

此功能应适用于您可能找到的任何版本的Elasticsearch的远程群集。这应该允许您从任何版本的Elasticsearch升级到当前版本，通过从旧版本的集群重新建立索引。

要启用发送到旧版本Elasticsearch的查询，`query`参数将直接发送到远程主机，无需验证或修改。

来自远程服务器的重新索引使用默认为最大大小为`100mb`的堆栈缓冲区。如果远程索引包含非常大的文档，则需要使用较小的批量大小。下面的示例设置非常非常小的批量大小`10`。

```js
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200"
    },
    "index": "source",
    "size": 10,
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

也可以使用`socket_timeout`字段在远程连接上设置`socket`的读取超时，并使用`connect_timeout`字段设置连接超时。两者默认为三十秒。此示例将套接字读取超时设置为一分钟，并将连接超时设置为十秒：

```js
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "socket_timeout": "1m",
      "connect_timeout": "10s"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

## URL参数

除了标准参数像`pretty`之外，“Reindex API”还支持`refresh`、`wait_for_completion`、`wait_for_active_shards`、`timeout`以及`requests_per_second`。

发送`refresh`将在更新请求完成时更新索引中的所有分片。这不同于 Index API 的`refresh`参数，只会导致接收到新数据的分片被索引。

如果请求包含`wait_for_completion=false`，那么Elasticsearch将执行一些预检检查、启动请求、然后返回一个任务，可以与[Tasks API](#docs-delete-by-query-task-api)一起使用来取消或获取任务的状态。Elasticsearch还将以`.tasks/task/${taskId}`作为文档创建此任务的记录。这是你可以根据是否合适来保留或删除它。当你完成它时，删除它可以让Elasticsearch回收它使用的空间。

`wait_for_active_shards`控制在继续请求之前必须有多少个分片必须处于活动状态，详见[这里](./Index_API.md#index-wait-for-active-shards)。`timeout`控制每个写入请求等待不可用分片变成可用的时间。两者都能正确地在[Bulk API](./Bulk_API.md)中工作。

`requests_per_second`可以设置为任何正数（1.4，6，1000等），来作为“delete-by-query”每秒请求数的节流阀数字，或者将其设置为`-1`以禁用限制。节流是在批量批次之间等待，以便它可以操纵滚动超时。等待时间是批次完成的时间与`request_per_second * requests_in_the_batch`的时间之间的差异。由于分批处理没有被分解成多个批量请求，所以会导致Elasticsearch创建许多请求，然后等待一段时间再开始下一组。这是“突发”而不是“平滑”。默认值为-1。

## 响应体

JSON响应类似如下：

```js
{
  "took" : 639,
  "updated": 0,
  "created": 123,
  "batches": 1,
  "version_conflicts": 2,
  "retries": {
    "bulk": 0,
    "search": 0
  }
  "throttled_millis": 0,
  "failures" : [ ]
}
```

`took`

    从整个操作的开始到结束的毫秒数。

`updated`

    成功更新的文档数。

`upcreateddated`

    成功创建的文档数。

`batches`

    通过查询更新的滚动响应数量。

`version_conflicts`

    根据查询更新时，版本冲突的数量。

`retries`

    根据查询更新的重试次数。bluk 是重试的批量操作的数量，search 是重试的搜索操作的数量。

`throttled_millis`

    请求休眠的毫秒数，与`requests_per_second`一致。

`failures`

    失败的索引数组。如果这是非空的，那么请求因为这些失败而中止。请参阅 conflicts 来如何防止版本冲突中止操作。

## 配合Task API使用

您可以使用[Task API](../Cluster_APIs/Task_Management_API.md)获取任何正在运行的重建索引请求的状态：

```js
GET _tasks?detailed=true&actions=*/update/byquery
```

响应会类似如下：

```js
{
  "nodes" : {
    "r1A2WoRbTwKZ516z6NEs5A" : {
      "name" : "r1A2WoR",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1:9300",
      "attributes" : {
        "testattr" : "test",
        "portsfile" : "true"
      },
      "tasks" : {
        "r1A2WoRbTwKZ516z6NEs5A:36619" : {
          "node" : "r1A2WoRbTwKZ516z6NEs5A",
          "id" : 36619,
          "type" : "transport",
          "action" : "indices:data/write/reindex",
          "status" : {    //①
            "total" : 6154,
            "updated" : 3500,
            "created" : 0,
            "deleted" : 0,
            "batches" : 4,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": {
              "bulk": 0,
              "search": 0
            },
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}
```

① 此对象包含实际状态。它就像是响应json，重要的添加`total`字段。 `total`是重建索引希望执行的操作总数。您可以通过添加的`updated`、`created`和`deleted`的字段来估计进度。当它们的总和等于`total`字段时，请求将完成。

使用任务id可以直接查找任务：

```js
GET /_tasks/taskId:1
```

这个API的优点是它与`wait_for_completion=false`集成，以透明地返回已完成任务的状态。如果任务完成并且`wait_for_completion=false`被设置，那么它将返回`results`或`error`字段。此功能的成本是`wait_for_completion=false`在`.tasks/task/${taskId}`创建的文档，由你自己删除该文件。

## 配合取消任务API使用

所有重建索引都能使用[Task Cancel API](../Cluster_APIs/Task_Management_API.md)取消：

```js
POST _tasks/task_id:1/_cancel
```

可以使用上面的任务API找到`task_id`。

取消应尽快发生，但可能需要几秒钟。上面的任务状态API将继续列出任务，直到它被唤醒取消自身。

## 重置节流阀

`request_per_second`的值可以在通过查询删除时使用`_rethrottle` API更改：

```js
POST _update_by_query/task_id:1/_rethrottle?requests_per_second=-1
```

可以使用上面的任务API找到task_id。 

就像在`_update_by_query` API中设置它一样，`request_per_second`可以是`-1`来禁用限制，或者任何十进制数字，如1.7或12，以节制到该级别。加速查询的会立即生效，但是在完成当前批处理之后，减慢查询的才会生效。这样可以防止滚动超时。

## <span id="docs-reindex-change-name">修改字段名</span>

`_reindex`可用于使用重命名的字段构建索引的副本。假设您创建一个包含如下所示的文档的索引：

```js
POST test/test/1?refresh
{
  "text": "words words",
  "flag": "foo"
}
```

但是你不喜欢这个`flag`名称，而是要用`tag`替换它。 `_reindex`可以为您创建其他索引：

```js
POST _reindex
{
  "source": {
    "index": "test"
  },
  "dest": {
    "index": "test2"
  },
  "script": {
    "inline": "ctx._source.tag = ctx._source.remove(\"flag\")"
  }
}
```

现在你可以得到新的文件：

```js
GET test2/test/1
```

它看起来像：

```js
{
  "found": true,
  "_id": "1",
  "_index": "test2",
  "_type": "test",
  "_version": 1,
  "_source": {
    "text": "words words",
    "tag": "foo"
  }
}
```

或者你可以通过`tag`进行任何你想要的搜索。

## 手动切片

重建索引支持[滚动切片](../Search_APIs/Request_Body_Search/Scroll.md#sliced-scroll)，您可以相对轻松地手动并行化处理：

```js
POST _reindex
{
  "source": {
    "index": "twitter",
    "slice": {
      "id": 0,
      "max": 2
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
POST _reindex
{
  "source": {
    "index": "twitter",
    "slice": {
      "id": 1,
      "max": 2
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

您可以通过以下方式验证：

```js
GET _refresh
POST new_twitter/_search?size=0&filter_path=hits.total
```

其结果一个合理的`total`像这样：

```js
{
  "hits": {
    "total": 120
  }
}
```

## 自动切片

你还可以让重建索引使用切片的`_uid`来自动并行的[滚动切片](../Search_APIs/Request_Body_Search/Scroll.md#sliced-scroll)。

```js
POST _reindex?slices=5&refresh
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

您可以通过以下方式验证：

```js
POST new_twitter/_search?size=0&filter_path=hits.total
```

其结果一个合理的`total`像这样：

```js
{
  "hits": {
    "total": 120
  }
}
```

将`slices`添加到`_reindex`中可以自动执行上述部分中使用的手动过程，创建子请求，这意味着它有一些怪癖：

- 您可以在[Task API](#docs-delete-by-query-task-api)中看到这些请求。这些子请求是具有`slices`请求任务的“子”任务。
- 获取`slices`请求任务的状态只包含已完成切片的状态。
- 这些子请求可以单独寻址，例如取消和重置节流阀。
- `slices`的重置节流阀请求将按相应的重新计算未完成的子请求。
- `slices`的取消请求将取消每个子请求。
- 由于`slices`的性质，每个子请求将不会获得完全均匀的文档部分。所有文件都将被处理，但有些片可能比其他片大。预期更大的切片可以有更均匀的分布。
- 带有`slices`请求的`request_per_second`和`size`的参数相应的分配给每个子请求。结合上述关于分布的不均匀性，您应该得出结论，使用切片大小可能不会导致正确的大小文档为`_reindex`。
- 每个子请求都会获得源索引的略有不同的快照，尽管这些都是大致相同的时间。

## 挑选切片数量

在这一点上，我们围绕要使用的`slices`数量提供了一些建议（比如手动并行化时，切片API中的`max`参数）：

- 不要使用大的数字，`500`就能造成相当大的CPU抖动。
- 从查询性能的角度来看，在源索引中使用分片数量的一些倍数更为有效。
- 在源索引中使用完全相同的分片是从查询性能的角度来看效率最高的。
- 索引性能应在可用资源之间以`slices`数量线性扩展。
- 索引或查询性能是否支配该流程取决于许多因素，如正在重建索引的文档和进行`reindexing`的集群。

## 索引的日常重建

您可以使用`_reindex`与[Painless](../Modules/Scripting/Painless_Scripting_Language.md)组合来重新每日编制索引，以将新模板应用于现有文档。 假设您有由以下文件组成的索引：

```js
PUT metricbeat-2016.05.30/beat/1?refresh
{"system.cpu.idle.pct": 0.908}
PUT metricbeat-2016.05.31/beat/1?refresh
{"system.cpu.idle.pct": 0.105}
```

`metricbeat-*`索引的新模板已经加载到Elaticsearch中，但它仅适用于新创建的索引。Painless可用于重新索引现有文档并应用新模板。

下面的脚本从索引名称中提取日期，并创建一个附带有`-1`的新索引。来自`metricbeat-2016.05.31`的所有数据将重新索引到`metricbeat-2016.05.31-1`。

```js
POST _reindex
{
  "source": {
    "index": "metricbeat-*"
  },
  "dest": {
    "index": "metricbeat"
  },
  "script": {
    "lang": "painless",
    "inline": "ctx._index = 'metricbeat-' + (ctx._index.substring('metricbeat-'.length(), ctx._index.length())) + '-1'"
  }
}
```

来自上一个度量索引的所有文档现在可以在`*-1`索引中找到。

```js
GET metricbeat-2016.05.30-1/beat/1
GET metricbeat-2016.05.31-1/beat/1
```

以前的方法也可以与[更改字段的名称](#docs-reindex-change-name)一起使用，以便将现有数据加载到新索引中，但如果需要，还可以重命名字段。

## 提取索引的随机子集

Reindex可用于提取用于测试的索引的随机子集：

```js
POST _reindex
{
  "size": 10,
  "source": {
    "index": "twitter",
    "query": {
      "function_score" : {
        "query" : { "match_all": {} },
        "random_score" : {}
      }
    },
    "sort": "_score"    //①
  },
  "dest": {
    "index": "random_twitter"
  }
}
```

① Reindex默认按`_doc`排序，所以`random_score`不会有任何效果，除非您将排序重写为`_score`。

