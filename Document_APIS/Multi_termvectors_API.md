# Multi termvectors API

Multi termvectors API允许一次获得多个词条向量。检索词条向量的文档由索引
、类型和ID指定。但文件也可以在请求本身中人为地提供。 响应包括一个具有所有获取的术语的`docs`数组，每个元素具有由[termvectors](./Term_Vectors.md) API提供的结构。这是一个例子：

```js
POST /_mtermvectors
{
   "docs": [
      {
         "_index": "twitter",
         "_type": "tweet",
         "_id": "2",
         "term_statistics": true
      },
      {
         "_index": "twitter",
         "_type": "tweet",
         "_id": "1",
         "fields": [
            "message"
         ]
      }
   ]
}
```

有关可能的参数的描述，请参阅[termvectors](./Term_Vectors.md) API。

`_mtermvectors`端点也可以针对索引使用（在这种情况下，它不需要在主体中）：

```js
POST /twitter/_mtermvectors
{
   "docs": [
      {
         "_type": "tweet",
         "_id": "2",
         "fields": [
            "message"
         ],
         "term_statistics": true
      },
      {
         "_type": "tweet",
         "_id": "1"
      }
   ]
}
```

以及类型：

```js
POST /twitter/tweet/_mtermvectors
{
   "docs": [
      {
         "_id": "2",
         "fields": [
            "message"
         ],
         "term_statistics": true
      },
      {
         "_id": "1"
      }
   ]
}
```

如果所有请求的文档都在相同的索引并且具有相同的类型，并且参数是相同的，则可以简化请求：

```js
POST /twitter/tweet/_mtermvectors
{
    "ids" : ["1", "2"],
    "parameters": {
        "fields": [
                "message"
        ],
        "term_statistics": true
    }
}
```

此外，就像对于[termvectors](./Term_Vectors.md) API一样，可以为用户提供的文档生成词条向量。所使用的映射由`_index`和`_type`确定。

```js
POST /_mtermvectors
{
   "docs": [
      {
         "_index": "twitter",
         "_type": "tweet",
         "doc" : {
            "user" : "John Doe",
            "message" : "twitter test test test"
         }
      },
      {
         "_index": "twitter",
         "_type": "test",
         "doc" : {
           "user" : "Jane Doe",
           "message" : "Another twitter test ..."
         }
      }
   ]
}
```