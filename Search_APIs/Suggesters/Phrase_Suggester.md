# Phrase Suggester

> 注意
>
> 为了理解 suggestions 的形式，请先阅读[suggesters](../Suggesters.md)。

术语 suggester 提供了一种非常方便的 API ，以在某个字符串距离内在每个 token 的基础上访问字替换。 API 允许单独访问流中的每个 token ，而 suggest 选择由API使用者选择。 然而，通常需要预先选择的 suggestions 以呈现给最终用户。 短语 suggester 在 term suggester 之上添加额外的逻辑以选择整个经校正的短语，而不是基于 ngram-language 模型加权的单个 token 。 在实践中，这个 suggester 将能够基于共现和频率来做出关于选择哪些 token 的更好的决定。

## API 示例

一般来说，phrase suggester  需要前面的特殊映射。 此页面上的 phrase suggester 示例需要以下映射才能正常工作。 仅在最后一个示例中使用反向(reverse)分析器。

```bash
PUT test
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "analysis": {
        "analyzer": {
          "trigram": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["standard", "shingle"]
          },
          "reverse": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["standard", "reverse"]
          }
        },
        "filter": {
          "shingle": {
            "type": "shingle",
            "min_shingle_size": 2,
            "max_shingle_size": 3
          }
        }
      }
    }
  },
  "mappings": {
    "test": {
      "properties": {
        "title": {
          "type": "text",
          "fields": {
            "trigram": {
              "type": "text",
              "analyzer": "trigram"
            },
            "reverse": {
              "type": "text",
              "analyzer": "reverse"
            }
          }
        }
      }
    }
  }
}
POST test/test?refresh=true
{"title": "noble warriors"}
POST test/test?refresh=true
{"title": "nobel prize"}
```

一旦你设置了分析器和映射，你可以在同一个地方使用 phrase suggester，你可以使用 term suggester ：

```bash
POST test/_search
{
  "suggest": {
    "text": "noble prize",
    "simple_phrase": {
      "phrase": {
        "field": "title.trigram",
        "size": 1,
        "gram_size": 3,
        "direct_generator": [ {
          "field": "title.trigram",
          "suggest_mode": "always"
        } ],
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```

该响应包含由最可能的拼写纠正评分的 suggestions 。 在这种情况下，我们收到了预期的校正“诺贝尔奖”。

```bash
{
  "_shards": ...
  "hits": ...
  "timed_out": false,
  "took": 3,
  "suggest": {
    "simple_phrase" : [
      {
        "text" : "noble prize",
        "offset" : 0,
        "length" : 11,
        "options" : [ {
          "text" : "nobel prize",
          "highlighted": "<em>nobel</em> prize",
          "score" : 0.5962314
        }]
      }
    ]
  }
}
```

## 基本短语 suggest API 参数

参数          | 描述
---------|-------------------
`field`         | 用于对语言模型进行n元语法查找的字段的名称， suggester 将使用此字段获取统计信息以对校正进行评分。 此字段是必填字段。
`gram_size` | 在字段中设置 n-gram（shingles）的最大大小。 如果字段不包含 n-gram（shingles），则应省略或设置为1.请注意，Elasticsearch 尝试根据指定的字段检测克大小。 如果字段使用 shingle 过滤器，则如果未明确设置，则将gram_size 设置为 max_shingle_size。
`real_word_error_likelihood` | 即使词语存在于字典中，词语是拼写错误的可能性。 默认是0.95对应5％的真实单词拼写错误。
`confidence` | 置信水平定义了应用于输入短语分数的因子，其被用作其他 suggest 候选的阈值。 只有得分高于阈值的候选人才会包括在结果中。 例如，1.0的置信水平将仅返回得分高于输入短语的 suggestions 。 如果设置为0.0，则返回前N个候选。 默认值为1.0。
`max_errors` | 为了形成校正，最多被认为是拼写错误的术语的最大百分比。 此方法接受范围[0..1]中的浮点值作为实际查询项的分数或作为查询项的绝对数量的数字> = 1。 默认值设置为1.0，对应于只返回最多1个拼写错误项的更正。 请注意，将其设置过高可能会对性能产生负面影响。 推荐使用低值，例如1或2，否则 suggestions 调用的时间花费可能超过查询执行的时间花费。
`separator` | 用于分隔 bigram 字段中的术语的分隔符。 如果未设置，则使用空格字符作为分隔符
`size`         | 为每个单独查询项生成的候选数量低数字（如3或5）通常会产生良好的结果。 提高这个可以带来更高的编辑距离的术语。 默认值为5。
`analyzer`   | 将分析器设置为分析以使用 suggest 文本。 默认为通过字段传递的 suggest 字段的搜索分析器。
`shard_size` | 设置要从每个单独的分片检索的 suggestions 字词的最大数量。 在减少阶段期间，基于size选项只返回前N个 suggestions 。 默认为5。
`text`         | 设置文本/查询以提供 suggestions 。
`highlight`  | 设置 suggestion 高亮显示。 如果未提供，则不返回高亮显示的字段。 如果提供，必须包含完全pre_tag和post_tag包裹改变的标记。 如果一行中的多个标记被改变，则改变的标记的整个短语被包装，而不是每个标记。
`collate`     | 检查针对指定查询的每个 suggestion ，以修剪索引中没有匹配的文档的 suggestions 。 对于 suggestion 的整理查询仅在从中生成 suggestion 的本地碎片上运行。 必须指定查询，并将其作为模板查询运行。 当前 suggestion 自动提供为 {{suggestion}}} 变量，应在您的查询中使用。 您仍然可以指定自己的模板 params-suggestions 值将添加到您指定的变量。 此外，您可以指定一个 prune 以控制是否返回所有短语 suggestions ，设置为 true时， suggestions 将有一个附加选项 collate_match，如果找到匹配的短语文档，则为true，否则为false。 prune 的默认值为false。

```bash
POST _search
{
  "suggest": {
    "text" : "noble prize",
    "simple_phrase" : {
      "phrase" : {
        "field" :  "title.trigram",
        "size" :   1,
        "direct_generator" : [ {
          "field" :            "title.trigram",
          "suggest_mode" :     "always",
          "min_word_length" :  1
        } ],
        "collate": {
          "query": { # ①
            "inline" : {
              "match": {
                "{{field_name}}" : "{{suggestion}}"  # ②
              }
            }
          },
          "params": {"field_name" : "title"},  # ③
          "prune": true # ④
        }
      }
    }
  }
}
```
① 所有这三个元素都是可选的。
____________________________
② {{suggestion}} 变量将会被每个 suggestion 文本所代替。
____________________________
③ 额外的 field_name 变量已经在 params 中被指定，并且被 match 查询所使用。
____________________________
④  所有的 suggestions 将和一个额外的 collate_match 选项一起返回，表示是否生成的短语匹配任何文档。
____________________________

## 平滑（smothing）模型

短语 suggester 支持多个平滑模型来平衡重量不频繁`grams`（grams（瓦）不存在于索引中）和频繁 `grams`（在索引中至少出现一次）。

`stupid_backoff` 简单的回退模型，如果高阶计数为0并且通过常数因子折扣低阶`n-gram`模型，则回退到低阶`n-gram`模型。 默认折扣为`0.4`。 `Stupid Backoff`是默认模型。
____________________________
`laplace` 使用添加平滑的平滑模型，其中将常数（通常为1.0或更小）添加到所有计数以平衡权重。默认`α`为`0.5`。
____________________________
`linear_interpolation` 平滑模型，其基于用户提供的权重（lambdas）获取单字组，双字组和三字母组的加权平均值。 线性插值没有任何默认值。 必须提供所有参数（`trigram_lambda`，`bigram_lambda`，`unigram_lambda`）。

## 候选生成器（ Generators）

短语 suggester 使用候选生成器来产生给定文本中每个术语的可能术语的列表。 单个候选生成器类似于对文本中的每个单独术语调用的术语 suggester 。 生成器的输出随后与来自其他项的候选一起被评分以用于 suggestions 候选。

目前只支持一种类型的候选生成器，`direct_generator`。 Phrase suggestions API接受在关键`direct_generator`下的生成器列表，列表中的每个生成器在原始文本中被称为每个term。

## 直接生成器（Generators）

直接生成器支持以下参数：

参数          | 描述
---------|-------------------
`field` | 从中获取候选 suggestions 的字段。 这是必需的选项，需要设置全局或每个 suggestion 。
`size` | 每个 suggestion 文本标记返回的最大更正值。
`suggest_mode` | suggest 模式控制在每个分片上生成的 suggestions 中包括哪些 suggestions 。 除了always之外的所有值都可以被认为是优化以生成更少的 suggestions 以在每个碎片上测试，并且在组合在每个碎片上生成的 suggestions 时不被重新检查。 因此，对于不包含它们的分片，即使其他分片包含它们，也会生成对分片的 suggestions 。 这些应该使用信心过滤掉。 可以指定三个可能的值：<br/>`missing:` 仅生成不在f中的术语的 suggestions 。 这是默认值。<br/>`popular`： 只 suggest 在 shard 的更多文档中出现的术语，而不是原始术语<br/>`always`：根据 suggestions 文字中的字词 suggest 任何相符的 suggestions 。
`max_edits` | 最大编辑距离候选 suggestions 可以具有，以便被认为是 suggestion 。 只能是介于1和2之间的值。任何其他值都会导致抛出错误的请求错误。 默认为2。
`prefix_length` | 必须匹配的最小前缀字符的数量是候选 suggestions 。 默认值为1.增加此数字可提高拼写检查性能。 通常拼写错误不会出现在术语的开头。 （旧名称“prefix_len”已弃用）
`min_word_length` | suggest 文本术语必须包含的最小长度。 默认值为4.（旧名称“min_word_len”已弃用）
`max_inspections` | 用于乘以 shards_size 以便在碎片级别上检查更多候选拼写校正的因子。 可以以性能为代价提高精度。 默认为5。
`min_doc_freq` | suggestion 应该出现的文档数量的最小阈值。这可以指定为绝对数字或文档数量的相对百分比。 这可以通过仅提示高频项来提高质量。 默认值为0f，未启用。 如果指定的值大于1，则该数字不能为小数。 分片级文档频率用于此选项。
`max_term_freq` | suggestion 文本标记可以存在的文档数量中的最大阈值，以便包括。 可以是表示文档频率的相对百分比数字（例如0.4）或绝对数字。 如果指定的值大于1，则不能指定小数。 默认为0.01f。 这可以用于排除高频术语的拼写检查。 高频项通常拼写正确，这也提高了拼写检查的性能。 分片级文档频率用于此选项。
`pre_filter` | 应用于传递到该候选生成器的每个令牌的过滤器（分析器）。 在生成候选项之前，此过滤器应用于原始令牌。
`post_filter` | 在它们被传递给实际短语记分器之前应用于每个生成的令牌的过滤器（分析器）。

他下面的例子显示了一个短语 suggest 用两个生成器调用，第一个使用包含普通索引术语的字段，第二个使用使用索引与反向过滤器的术语的字段（令牌是相反顺序的索引）。 这用于克服直接发电机的限制，需要恒定的前缀来提供高性能 suggestions 。 `pre_filter`和`post_filter`选项接受普通分析器名称。

```bash
POST _suggest
{
  "text" : "obel prize",
  "simple_phrase" : {
    "phrase" : {
      "field" : "title.trigram",
      "size" : 1,
      "direct_generator" : [ {
        "field" : "title.trigram",
        "suggest_mode" : "always"
      }, {
        "field" : "title.reverse",
        "suggest_mode" : "always",
        "pre_filter" : "reverse",
        "post_filter" : "reverse"
      } ]
    }
  }
}
```

`pre_filter`和`post_filter`也可以用于在生成候选项之后注入同义词。 例如，对于查询`caption usq`，我们可以为项`usq`生成候选`usa`，这是 `america`的同义词，其允许如果该短语得分足够高则向用户呈现`captain america`。