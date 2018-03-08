# Inner hits

[父/子](../../Mapping/Meta-Fields/_parent_field.md)和[嵌套](../..//Mapping/Field_datatypes/Nested_datatype.md)功能允许返回在不同范围内具有匹配的文档。 在父/子情况下，基于子文档中的匹配返回父文档，或者基于父文档中的匹配返回子文档。 在嵌套的情况下，基于嵌套内部对象中的匹配返回文档。

在这两种情况下，导致返回文档的不同作用域中的实际匹配都被隐藏。 在许多情况下，知道哪些内部嵌套对象（在嵌套的情况下）或子/父文档（在父/子）的情况下导致返回某些信息是非常有用的。 内部命中功能可用于此。 此功能会在搜索响应中返回每个搜索匹配的附加嵌套匹配，导致搜索匹配在不同范围内匹配。

可以通过在嵌套，`has_child`或`has_parent`查询和过滤器上定义`inner_hits`定义来使用内部命中。 结构如下所示：

```js
"<query>" : {
    "inner_hits" : {
        <inner_hits_options>
    }
}
```

如果在支持它的查询上定义`inner_hits`，则每个搜索命中将包含具有以下结构的`inner_hits`json对象：

```json
"hits": [
    {
    "_index": ...,
    "_type": ...,
    "_id": ...,
    "inner_hits": {
        "<inner_hits_name>": {
            "hits": {
                "total": ...,
                "hits": [
                {
                    "_type": ...,
                    "_id": ...,
                    ...
                },
                ...
                ]
            }
        }
    },
    ...
    },
    ...
]
```
 
## Options

内部命中支持以下选项：

选项        | 描述
--------|----------
`from`       | 从返回的常规搜索命中的每个 inner_hits 的第一个命中的获取位置的偏移量。
`size`         | 每个 inner_hits 返回的命中的最大数量。 默认情况下，返回前三个命中匹配。
`sort`         | 如何内部命中应该按inner_hits排序。 默认情况下，匹配按分数排序。
`name`       | 在响应中用于特定内部命中定义的名称。 在单个搜索请求中定义多个内部命中时非常有用。 默认值取决于定义内部命中的查询。 对于`has_child`查询和过滤器，这是子类型，对于`has_parent`查询和过滤器，这是父类型和嵌套查询，对于过滤器，这是嵌套路径。

内部命中还支持以下每个文档功能：

- [Highlighting](./Highlighting.md)
- [Explain](./Explain.md)
- [Source filtering](./Source_filtering.md)
- [Script fields](./Script_Fields.md)
- [Doc value fields](./Doc_value_Fields.md)
- [Include versions](./Version.md)

## Nested inner hits

嵌套`inner_hits`可用于将嵌套内部对象包含为搜索匹配的内部匹配。

下面的示例假设有一个使用名称`comment`定义的嵌套对象字段：

```js
 {
    "query" : {
        "nested" : {
            "path" : "comments",
            "query" : {
                "match" : {"comments.message" : "[actual query]"}
            },
            "inner_hits" : {}    //①
        }
    }
}
```

① 嵌套查询中的内部命中定义。 没有其他选项需要定义。
________________________________


一个可以从上述搜索请求生成的响应片段的示例：

```js
 ...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comments": { ①
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_nested": {
                          "field": "comments",
                          "offset": 2
                       },
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```

① 搜索请求中的内部匹配定义中使用的名称。 可以通过名称选项使用自定义键。
________________________________
 

在上面的例子中，`_nested `元数据是至关重要的，因为它定义了这个内部命中来自什么内部嵌套对象。 该字段定义嵌套命中来自的对象数组字段，以及相对于其在`_source`中的位置的偏移。 由于排序和评分，命中对象在`inner_hits`中的实际位置通常不同于定义嵌套内部对象的位置。

默认情况下，对于`inner_hits`中的命中对象也返回`_source`，但这可以更改。 通过`_source`过滤功能部分源可以返回或禁用。 如果存储字段在嵌套级别上定义，则也可以通过字段特性返回。

一个重要的缺省是在`inner_hits`内的`hits`中返回的`_source`是相对于`_nested`元数据。 因此，在上面的示例中，仅对每个嵌套的匹配返回注释部分，而不是包含注释的顶级文档的整个源。

## Hierarchical levels of nested object fields and inner hits

如果映射具有多层次的层次化嵌套对象字段，则每个层次都可以通过点标记路径访问。 例如，如果存在包含票据嵌套字段的注释嵌套字段，并且应该直接使用根命中返回票，则可以定义以下路径：

```js
 {
   "query" : {
      "nested" : {
         "path" : "comments.votes",
         "query" : { ... },
         "inner_hits" : {}
      }
    }
}
```

此间接引用仅支持嵌套内部命中。



## Parent/child inner hits

父/子`inner_hits`可以用于包括父或子

以下示例假定在注释类型中存在`_parent`字段映射：

```js
 {
    "query" : {
        "has_child" : {
            "type" : "comment",
            "query" : {
                "match" : {"message" : "[actual query]"}
            },
            "inner_hits" : {}       // ①
        }
    }
}
```

①  内部命中定义类似于嵌套示例。
________________________________
 

一个可以从上述搜索请求生成的响应片段的示例：

```js
 ...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comment": {
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": "comment",
                       "_id": "5",
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```