# Completion Suggester

> 注意
>
> 为了理解 suggestions 的形式，请先阅读[suggesters](../Suggesters.md)。

`completion` suggester 提供自动完成/按需搜索功能。 这是一种导航功能，可在用户输入时引导用户查看相关结果，从而提高搜索精度。 它不是用于拼写校正或平均值功能，如`term`或`phrase` suggesters.

理想地，自动完成功能应当与用户键入的速度一样快，以提供与用户已经键入的内容相关的即时反馈。因此，完成 suggester 针对速度进行优化。  suggester 使用允许快速查找的数据结构，但是构建成本高并且存储在存储器中。

## 映射（mapping）

要使用此功能，请为此字段指定一个特殊映射，为快速完成的字段值编制索引。

```bash
PUT music
{
    "mappings": {
        "song" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion"
                },
                "title" : {
                    "type": "keyword"
                }
            }
        }
    }
}
```

映射支持以下参数：

- `analyzer`
    - 使用索引分析器，默认为简单。 如果你想知道为什么我们没有选择标准分析器：我们尝试在这里很容易理解的行为，如果你索引字段内容在`Drive-in`，你不会得到任何建议， （第一个非停用词）

- `search_analyzer`
    - 要使用的搜索分析器，默认为分析器的值。

- `preserve_separators`
    - 保留分隔符，默认为`true`。 如果禁用，你可以找到一个以`Foo Fighters`开头的字段，如果你推荐`foof`。

- preserve_position_increments
    - 启用位置增量，默认为`true`。 如果禁用和使用停用分析器，您可以得到一个字段从披头士开始，如果你`suggest b`。 注意：你也可以通过索引两个输入，`Beatles`和`The Beatles`，不需要改变一个简单的分析器，如果你能够丰富你的数据。

- `max_input_length`
    - 限制单个输入的长度，默认为`50`个UTF-16代码点。 此限制仅在索引时使用，以减少每个输入字符串的字符总数，以防止大量输入膨胀底层数据结构。 大多数用例不会受默认值的影响，因为前缀完成很少超过前缀长度超过少数几个字符。

## 索引

您像任何其他字段一样索引 suggestion 。 suggestion 由输入和可选的权重属性组成。 输入是要由 suggestion 查询匹配的期望文本，并且权重确定如何对 suggestion 进行评分。 索引 suggestion 如下：

```bash
PUT music/song/1?refresh
{
    "suggest" : {
        "input": [ "Nevermind", "Nirvana" ],
        "weight" : 34
    }
}
```

以下参数被支持：

- `input`
    - 输入存储，这可以是字符串数组或只是一个字符串。 此字段是必填字段。

- `weight`
    - 正整数或包含正整数的字符串，用于定义权重并允许对 suggestions 进行排名。 此字段是可选的。

您可以按如下所示为文档编制多个 suggestion s：

```bash
PUT music/song/1?refresh
{
    "suggest" : [
        {
            "input": "Nevermind",
            "weight" : 10
        },
        {
            "input": "Nirvana",
            "weight" : 3
        }
    ]
}
```

您可以使用以下速记形式。 请注意，您不能使用 suggestion 指定权重。

```bash
PUT music/song/1?refresh
{
  "suggest" : [ "Nevermind", "Nirvana" ]
}
```
 
## 查询
suggest 像往常一样工作，除了您必须指定 suggest 类型为完成。 suggestions 接近实时，这意味着可以通过刷新显示新 suggestions ，并且一旦删除就不会显示文档。 此请求：

```bash
POST music/_suggest?pretty
{
    "song-suggest" : {
        "prefix" : "nir",
        "completion" : {
            "field" : "suggest"
        }
    }
}
```

返回这个响应：
```js
{
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "song-suggest" : [ {
    "text" : "nir",
    "offset" : 0,
    "length" : 3,
    "options" : [ {
      "text" : "Nirvana",
      "_index": "music",
      "_type": "song",
      "_id": "1",
      "_score": 1.0,
      "_source": {
        "suggest": ["Nevermind", "Nirvana"]
      }
    } ]
  } ]
}
```

> **重要**
>
> `_source`元字段必须启用，这是默认行为，以启用返回`_source`与 suggestions 。

配置的 suggestion 的权重返回为`_score`。 文本字段使用您的索引 suggestion 的输入。 默认情况下， suggestion 返回完整的文档`_source`。 由于磁盘读取和网络传输开销，`_source`的大小可能会影响性能。 为了节省一些网络开销，使用[源过滤](../Request_Body_Search/Source_filtering.md)从`_source`中过滤掉不必要的字段，以最小化`_source`大小。 请注意，`_suggest` 端点不支持源过滤，但在`_search`端点上使用 suggestion ：

```bash
POST music/_search?size=0
{
    "_source": "suggest",
    "suggest": {
        "song-suggest" : {
            "prefix" : "nir",
            "completion" : {
                "field" : "suggest"
            }
        }
    }
}

```
应该看起来像:

```js
{
    "took": 6,
    "timed_out": false,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    },
    "hits": {
        "total" : 0,
        "max_score" : 0.0,
        "hits" : []
    },
    "suggest": {
        "song-suggest" : [ {
            "text" : "nir",
            "offset" : 0,
            "length" : 3,
            "options" : [ {
                "text" : "Nirvana",
                "_index": "music",
                "_type": "song",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "suggest": ["Nevermind", "Nirvana"]
                }
            } ]
        } ]
    }
}
```

基本的完全 suggester 查询支持以下参数：

- `field`
    - 要运行查询的字段的名称（必需）

- `size`
    - 要返回的 suggestions 数（默认为`5`）。

> 注意
>
> 完全 suggester 考虑索引中的所有文档。 有关如何查询文档子集的解释，请参阅 Context Suggester。

> 注意
>
> 在跨越多个碎片的完成查询的情况下， suggest 在两个阶段中执行，其中最后阶段从碎片提取相关文档，这意味着对于单个碎片的执行完成请求由于文档提取开销而更加高效， suggest 跨越多个碎片。 为了获得最佳的完成性能， 建议将完成索引到单个分片索引中。 在由于碎片大小而导致堆使用率过高的情况下，仍建议将索引拆分为多个分片，而不是优化完成性能。

## 模糊查询

完成 suggester 还支持模糊查询 - 这意味着，您可以在搜索中输入错误，并仍然返回结果。

```bash
POST music/_suggest?pretty
{
    "song-suggest" : {
        "prefix" : "nor",
        "completion" : {
            "field" : "suggest",
            "fuzzy" : {
                "fuzziness" : 2
            }
        }
    }
}
```

与查询前缀共享最长前缀的 suggestion 将得分更高。

模糊查询可以采用特定的模糊参数。 支持以下参数：

参数 | 描述
----|----
`fuzziness` | 模糊系数，默认为AUTO。 有关允许的设置，请参阅 “Fuzzinessedit”一节。
`transpositions` | 如果设置为true，则换位计数为一个更改而不是两个，默认为true
`min_length` | 返回模糊 suggestions 前的输入的最小长度，默认值3
`prefix_length` | 输入的最小长度（未针对模糊替代项进行检查）默认为1
`unicode_aware` | 如果为true，则所有度量（如模糊编辑距离，置换和长度）都以Unicode代码点而不是字节为单位。 这比原始字节稍慢，因此默认情况下设置为false。

> 注意
>
> 如果你想坚持使用默认值，但仍然使用模糊，你可以使用  `fuzzy：{}`或`fuzzy：true`。

## 正则表达式查询

完成 suggester 还支持正则表达式查询，意味着您可以将前缀表达为正则表达式

```bash
POST music/_suggest?pretty
{
    "song-suggest" : {
        "regex" : "n[ever|i]r",
        "completion" : {
            "field" : "suggest"
        }
    }
}
```

正则表达式查询可以使用特定的正则表达式参数。 支持以下参数：

参数 | 描述
----|----
`flags` | 可能的标志是`ALL`（默认），`ANYSTRING`，`COMPLEMENT`，`EMPTY`，`INTERSECTION`，`INTERVA`L或`NONE`。 有关它们的含义，请参见[regexp-syntax](../..//Query_DSL/Term_level_queries/Regexp_Query.md).
`max_determinized_states` | 正则表达式是危险的，因为很容易意外地创建一个无害的，需要指数数量的内部确定的自动机状态（以及相应的RAM和CPU）执行 Lucene 。 Lucene使用`max_determinized_states`设置（默认为`10000`）阻止这些操作。 您可以提高此限制以允许执行更复杂的正则表达式。
 

