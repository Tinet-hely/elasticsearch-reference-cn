# Term Vectors（词条向量）

回有关特定文档字段中的词条的信息和统计信息。文档可以存储在索引中或由用户人工提供。词条向量默认为[实时](./Get_API.md#realtime)，不是近实时。这可以通过将`realtime`参数设置为`false`来更改。

```js
GET /twitter/tweet/1/_termvectors
```

可选的，您可以使用`url`中的参数指定检索信息的字段：

```js
GET /twitter/tweet/1/_termvectors?fields=message
```

或通过在请求主体中添加请求的字段（参见下面的示例）。也可以使用通配符指定字段，类似于[多匹配查询](../Query_DSL/Full_text_queries/Multi_Match_Query.md)

> 警告
>
> 请注意`/_termvector`的使用方式在2.0中已废弃，请使用`_termvectors`替代。

## 返回值

请求可以得到三种类型的值：词条信息，词条统计和字段统计。默认情况下，所有词条信息与字段统计信息都会被返回，但不包含词条统计信息。

### 词条信息

- 在字段中的词频（总是返回）
- 词条位置（`positions`: `true`）
- 开始与结束的偏移量（`offsets`: `true`）
- 词条有效载荷（`payloads`: `true`），base64编码的字节

如果请求的信息没有存储在索引中，如果可能它将被即时计算。另外，对于甚至不存在于索引中但由用户提供的文档，也可以计算词条向量。

> 警告
>
> 开始与结束的偏移量假设UTF-16编码被使用。如果要使用这些偏移量来从原始文本中获取词条，则应确保使用UTF-16对正在使用的子字符串进行编码。

### 词条统计

设置`term_statistics`为`true`（默认为`false`）将返回：

- 总词频（所有文件中的词条频率）
- 文档频率（包含词条的文档数）

默认情况下这些值不返回,因为词条统计数据会严重影响性能。

### 字段统计

将`field_statistics`设置为`false`（默认值为true）将省略：

- 文档数（包含此字段的文档数）
- 文档频率的总和（本字段中所有词条的文档频率的总和）
- 词频的总和（该字段中每个词条的词频的总和）

### 词条过滤

使用参数`filter`，返回的词条也可以根据其`tf-idf`分数进行过滤。这可能是有用的良好特征向量，以便找到文档。此功能的工作方式与[More Like This Query](../Query_DSL/Specialized_queries/More_Like_This_Query.md)的[第二章节](../Query_DSL/Specialized_queries/More_Like_This_Query.md#mlt-query-term-selection)相似。参见示[例5](#docs-termvectors-terms-filtering)的使用。 支持以下子参数：

参数名            | 描述
-----------------|-------------------
`max_num_terms`  | 每个字段必须返回的最大词条数。默认为`25`。
`min_term_freq`  | 在源文档中忽略少于此频率的单词。默认为`1`。
`max_term_freq`  | 在源文档中忽略超过此频率的单词。默认为无界。
`min_doc_freq`   | 忽略文档频率少于此参数的词条。默认为`1`。
`max_doc_freq`   | 忽略文档频率大于此参数的词条。默认为无界。
`min_word_length`| 字词长度低于此参数的将被忽略。默认为`0`。
`max_word_length`| 字词长度大于此参数的将被忽略。默认为无界（`0`）。

## 行为

词条和字段统计数据不准确。删除的文件不被考虑。这些信息只能用于所请求文档所在的分片。因此，术语和字段统计信息仅用作相对度量，而绝对数字在此上下文中无意义。默认情况下，当请求人造文档的词条向量时，随机选择获取统计信息的分片。使用`routing`将命中特定的分片。

### 示例：返回存储词条向量

首先，我们创建一个存储词条向量、有效载荷等的索引：

```js
PUT /twitter/
{ "mappings": {
    "tweet": {
      "properties": {
        "text": {
          "type": "text",
          "term_vector": "with_positions_offsets_payloads",
          "store" : true,
          "analyzer" : "fulltext_analyzer"
         },
         "fullname": {
          "type": "text",
          "term_vector": "with_positions_offsets_payloads",
          "analyzer" : "fulltext_analyzer"
        }
      }
    }
  },
  "settings" : {
    "index" : {
      "number_of_shards" : 1,
      "number_of_replicas" : 0
    },
    "analysis": {
      "analyzer": {
        "fulltext_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "filter": [
            "lowercase",
            "type_as_payload"
          ]
        }
      }
    }
  }
}
```

然后，我们添加一些文档：

```js
PUT /twitter/tweet/1
{
  "fullname" : "John Doe",
  "text" : "twitter test test test "
}

PUT /twitter/tweet/2
{
  "fullname" : "Jane Doe",
  "text" : "Another twitter test ..."
}
```

以下请求返回文档`1`（John Doe）中字段`text`的所有信息和统计信息：

```js
GET /twitter/tweet/1/_termvectors
{
  "fields" : ["text"],
  "offsets" : true,
  "payloads" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}
```

响应：

```js
{
    "_id": "1",
    "_index": "twitter",
    "_type": "tweet",
    "_version": 1,
    "found": true,
    "took": 6,
    "term_vectors": {
        "text": {
            "field_statistics": {
                "doc_count": 2,
                "sum_doc_freq": 6,
                "sum_ttf": 8
            },
            "terms": {
                "test": {
                    "doc_freq": 2,
                    "term_freq": 3,
                    "tokens": [
                        {
                            "end_offset": 12,
                            "payload": "d29yZA==",
                            "position": 1,
                            "start_offset": 8
                        },
                        {
                            "end_offset": 17,
                            "payload": "d29yZA==",
                            "position": 2,
                            "start_offset": 13
                        },
                        {
                            "end_offset": 22,
                            "payload": "d29yZA==",
                            "position": 3,
                            "start_offset": 18
                        }
                    ],
                    "ttf": 4
                },
                "twitter": {
                    "doc_freq": 2,
                    "term_freq": 1,
                    "tokens": [
                        {
                            "end_offset": 7,
                            "payload": "d29yZA==",
                            "position": 0,
                            "start_offset": 0
                        }
                    ],
                    "ttf": 2
                }
            }
        }
    }
}
```

### 示例：自动生成词条向量

未明确存储在索引中的词条向量将自动计算。以下请求返回文档`1`中字段的所有信息和统计信息，即使词条尚未明确存储在索引中。请注意，对于字段`text`，术语不会重新生成。

```js
GET /twitter/tweet/1/_termvectors
{
  "fields" : ["text", "some_field_without_term_vectors"],
  "offsets" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}
```

### 示例：人造文档

还可以为人造文档生成词条向量，也就是生成索引中不存在的文档。例如，以下请求将返回与示例1中相同的结果。所使用的映射由索引和类型确定。

如果动态映射打开（默认），则不在原始映射中的文档字段将被动态创建。

```js
GET /twitter/tweet/_termvectors
{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  }
}
```

#### Per-field 分析器

另外，可以通过使用`per_field_analyzer`参数来提供不同于当前的分析器。这对于以任何方式生成词条向量是有用的，特别是在使用人造文档时。当为已经存储的词条向量提供分析器时，将重新生成项向量。

```js
GET /twitter/tweet/_termvectors
{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  },
  "fields": ["fullname"],
  "per_field_analyzer" : {
    "fullname": "keyword"
  }
}
```

响应：

```js
{
  "_index": "twitter",
  "_type": "tweet",
  "_version": 0,
  "found": true,
  "took": 6,
  "term_vectors": {
    "fullname": {
       "field_statistics": {
          "sum_doc_freq": 2,
          "doc_count": 4,
          "sum_ttf": 4
       },
       "terms": {
          "John Doe": {
             "term_freq": 1,
             "tokens": [
                {
                   "position": 0,
                   "start_offset": 0,
                   "end_offset": 8
                }
             ]
          }
       }
    }
  }
}
```

### <span id="docs-termvectors-terms-filtering">示例：词条过滤</span>

最后，返回的词条可以根据他们的`tf-idf`分数进行过滤。在下面的例子中，我们从具有给定“plot”字段值的人造文档中获取三个“interesting”的关键字。请注意，关键字“Tony”或任何停止词不是响应的一部分，因为它们的`tf-idf`必须太低。

```js
GET /imdb/movies/_termvectors
{
    "doc": {
      "plot": "When wealthy industrialist Tony Stark is forced to build an armored suit after a life-threatening incident, he ultimately decides to use its technology to fight against evil."
    },
    "term_statistics" : true,
    "field_statistics" : true,
    "positions": false,
    "offsets": false,
    "filter" : {
      "max_num_terms" : 3,
      "min_term_freq" : 1,
      "min_doc_freq" : 1
    }
}
```

响应：

```js
{
   "_index": "imdb",
   "_type": "movies",
   "_version": 0,
   "found": true,
   "term_vectors": {
      "plot": {
         "field_statistics": {
            "sum_doc_freq": 3384269,
            "doc_count": 176214,
            "sum_ttf": 3753460
         },
         "terms": {
            "armored": {
               "doc_freq": 27,
               "ttf": 27,
               "term_freq": 1,
               "score": 9.74725
            },
            "industrialist": {
               "doc_freq": 88,
               "ttf": 88,
               "term_freq": 1,
               "score": 8.590818
            },
            "stark": {
               "doc_freq": 44,
               "ttf": 47,
               "term_freq": 1,
               "score": 9.272792
            }
         }
      }
   }
}
```