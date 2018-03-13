# Term suggester（词条建议器）

> 注意
>
> 为了理解 suggestions 的形式，请先阅读[suggesters](../Suggesters.md)。

`term`建议器根据编辑距离来进行词条建议。

## 常见的 suggest 选项

Options      |  Description
---------|------------------
`text`          | suggest 文本，suggest 文本是必须选项，需要被设定为全局或者对每个 suggestion。
`field`         | 从中获取候选 suggestions 的字段（field）。 这是必需的选项，需要设置为全局或按 suggestion 设置。
`analyzer`    |	分词器用来分析suggest文本，默认为 suggest 字段的分词器。
`size`          | 每个 suggest 文本标记（token）返回的最大更正值。
`sort`          | 定义每个 suggest 文本术语中 suggestions 该如何排序。 两个可能的值：<br/> `score`：先按照分数排序，然后按文档频率排序，然后是术语本身。<br/> `frequency`：按文档频率排序，然后依次选择相似性分数和术语本身。
`suggest_mode` | suggest_mode 控制什么 suggestions 被包括或控制什么 suggest 文本术语，什么 suggestions 应该被 suggested。 可以指定三个可能的值：<br/> `missing`： 只提供不在索引中的 suggest 文字字词的 suggestion 。 这是默认值。<br/> `popular`：只 suggest 出现在更多文档中的 suggestions，而不是原始 suggest 文本术语。<br/> `always`： 根据 suggest 文字中的字词 suggest 任何相符的 suggestions。

## 其它term suggest选项

Options      |  Description
---------|------------------
`lowercase_terms` | 在文本分析后的 小写 suggest 文本术语。
`max_edits` | 可以认为是候选 suggestions 的最大编辑距离。 只能是介于`1`和`2`之间的值。任何其他值都会导致抛出错误的请求错误。 默认为`2`。
`prefix_length` | 为了成为候选 suggestions 所必须匹配的最小前缀字符的数量。 默认值为`1`.增加此数字可提高拼写检查性能。 通常拼写错误不会出现在术语的开头。 （旧名称 “prefix_len” 已弃用）
`min_word_length` | suggest 文本术语必须包含的最小长度。 默认值为`4`.（旧名称 “min_word_len” 已弃用）
`shard_size` | 设置要从每个单独的分片检索的 suggestions 的最大数量。 在减少阶段期间，仅基于`size`选项返回前N个 suggestions。 默认为`size`选项。 将其设置为大于该`size`的值可以是有用的，以便以性能为代价获得更准确的拼写校正的文档频率。 由于术语在分片之间分割的事实，拼写校正的分片级文档频率可能不精确。 `
`max_inspections` | 用于乘以 shards_size 以便在碎片级别上检查更多候选拼写校正的因子。 可以以性能为代价提高精度。 默认为`5`。
`min_doc_freq` | suggestion 应该出现的文档数量的最小阈值。这可以指定为绝对数字或文档数量的相对百分比。 这可以通过仅 suggesting 高频项来提高质量。 默认值为`0f` ，未启用。 如果指定的值大于`1`，则该数字不能为小数。 分片级文档频率用于此选项。
`max_term_freq` | suggest 文本标记可以存在的文档数量中的最大阈值，以便包括。 可以是表示文档频率的相对百分比数字（例如0.4）或绝对数字。 如果指定的值大于1，则不能指定小数。 默认为 0.01f。 这可以用于排除高频术语的拼写检查。 高频项通常拼写正确，这也提高了拼写检查的性能。 分片级文档频率用于此选项。
`string_distance` | 使用哪个字符串距离实现来比较类似的 suggested 术语。 可以指定五个可能的值： internal - 基于 `damerau_levenshtein`的默认值，但是高度优化用于比较索引中的项的字符串距离。`damerau_levenshtein`—— 基于 Damerau-Levenshtein 算法的字符串距离算法。 `levenstein` —— 基于 Levenstein 编码距离算法的字符串距离算法。 `jarowinkler` —— 基于 Jaro-Winkler 算法的字符串距离算法。 `ngram` —— 基于字符`n-gram`的字符串距离算法。
