# Context Suggester

`completion` suggester 考虑索引中的所有文档，但通常希望提供通过某些标准过滤和/或提升的 suggestion 。 例如，您想要 suggestion 由某些艺术家过滤的歌曲标题，或者要根据其流派提高歌曲标题。

要实现 suggestion 过滤和/或提升，您可以在配置完成字段时添加上下文映射。 您可以为完成字段定义多个上下文映射。 每个上下文映射都有唯一的名称和类型。 有两种类型： `category`  和 `geo` 。 上下文映射在字段映射中的 `contexts` 参数下配置。

以下定义了类型，每个类型都有一个完成字段的两个上下文映射：

```bash
PUT place
{
    "mappings": {
        "shops" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion",
                    "contexts": [
                        { # ①
                            "name": "place_type",
                            "type": "category",
                            "path": "cat"
                        },
                        { # ②
                            "name": "location",
                            "type": "geo",
                            "precision": 4
                        }
                    ]
                }
            }
        }
    }
}
PUT place_path_category
{
    "mappings": {
        "shops" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion",
                    "contexts": [
                        { # ③
                            "name": "place_type",
                            "type": "category",
                            "path": "cat"
                        },
                        { # ④
                            "name": "location",
                            "type": "geo",
                            "precision": 4,
                            "path": "loc"
                        }
                    ]
                },
                "loc": {
                    "type": "geo_point"
                }
            }
        }
    }
}
```

① 定义名为 place_type 的 category 上下文，其中类别必须与 suggestions 一起发送。
_______________________
② 定义 geo context 名为 location，类别必须与 suggestions 一起发送。
_______________________
③ 定义名为 place_type 的 category 上下文，其中从cat字段读取类别。
_______________________
④ 定义 geo context 名为 location ，其中从 loc 字段读到 categories 。
_______________________

> 注意
>
> 添加上下文映射会增加完成字段的索引大小。 完成索引是完全堆驻留，您可以使用[Indices Stats](../../Indices_APIs/Indices_Stats.md)监视完成字段索引大小。

## 类别上下文（Category Context）

`category` context  允许您在索引时间将一个或多个类别与 suggestions 相关联。 在查询时，可以根据相关类别对 suggestions 进行过滤和提升。

映射设置为上面的 place_type 字段。 如果定义了路径，则从文档中的该路径读取类别，否则它们必须在 suggest 字段中发送，如下所示：

```bash
PUT place/shops/1
{
    "suggest": {
        "input": ["timmy's", "starbucks", "dunkin donuts"],
        "contexts": {
            "place_type": ["cafe", "food"]  # ①
        }
    }
}
```

① 这些 suggestions 将与 cafe 和 food 类别相关联。
_______________________

如果映射具有`path`，则以下索引请求将足以添加`categories`：

```bash
PUT place_path_category/shops/1
{
    "suggest": ["timmy's", "starbucks", "dunkin donuts"],
    "cat": ["cafe", "food"]  # ①
}
```

① 这些 suggestions 将与 cafe 和 food 类别相关联。
_______________________

如果上下文映射引用另一个字段，并且类别已明确编入索引，则 suggestions 将使用这两个类别进行索引。

## 类别查询

suggestions 可以按一个或多个类别进行过滤。 以下过滤了多个类别的 suggestions ：

```bash
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "place_type": [ "cafe", "restaurants" ]
            }
        }
    }
}
```

当在查询时未提供类别时，将考虑所有索引文档。 应避免在类别启用完成字段上没有类别的查询，因为它会降低搜索性能。

对某些类别的 suggestions 可以比其他类别更高。 以下内容按类别过滤 suggestions ，并增加与某些类别相关联的 suggestions ：

```bash
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "place_type": [  # ①
                    { "context" : "cafe" },
                    { "context" : "restaurants", "boost": 2 }
                 ]
            }
        }
    }
}
```

① 与类别咖啡馆和餐馆相关联的上下文查询过滤 suggestions ，并且将与餐馆相关联的 suggestions 提高2倍
_______________________

除了接受类别值之外，上下文查询可以由多个类别上下文子句组成。 类别上下文子句支持以下参数：

参数   | 描述
------|------
`context` | 要过滤/升级的类别的值。 这是强制性的。
`boost` | 应该提高 suggestion 的分数的因子，通过将增强乘以 suggestion 权重来计算分数，默认为1
`prefix` | 类别值是否应被视为前缀。 例如，如果设置为true，则可以通过指定类型的类别前缀来过滤类型1，类型2等的类别。 默认为false

## 地理位置上下文

地理位置上下文允许您将一个或多个地理位置或地理位置隐藏与 suggestions 在索引时间关联。 在查询时，如果 suggestions 在指定地理位置的某个距离内，则可以对 suggestions 进行过滤和提升。

在内部，地理点被编码为具有指定精度的 geohashes。

## 地理映射

除了路径设置，地理上下文映射接受以下设置：

参数   | 描述
------|------
`precision` | 这定义了要建立索引的 geohash 的精度，并且可以指定为距离值（5m，10km等）或原始 geohash 精度（1..12）。 默认为原始 geohash 精度值6。

> 注意
>
> 索引时间精度设置设置可在查询时使用的最大 geohash 精度。

## 索引地理上下文

地理上下文可以利用 suggestions 被显式地设置或者经由路径参数从文档中的地理点字段索引，类似于类别上下文。 将多个地理位置上下文与 suggestion 关联，将对每个地理位置的 suggestion 建立索引。 以下对具有两个地理位置上下文的 suggestion 进行索引：

```bash
PUT place/shops/1
{
    "suggest": {
        "input": "timmy's",
        "contexts": {
            "location": [
                {
                    "lat": 43.6624803,
                    "lon": -79.3863353
                },
                {
                    "lat": 43.6624718,
                    "lon": -79.3873227
                }
            ]
        }
    }
}
```

### 地理位置查询

suggestions 可以根据它们与一个或多个地理点的接近程度而被过滤和提升。 以下过滤 suggestions 落在由地理点的编码 geohash 表示的区域内：

```bash
POST place/_suggest
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "location": {
                    "lat": 43.662,
                    "lon": -79.380
                }
            }
        }
    }
}
```

当指定在查询时具有较低精度的位置时，将考虑落入该区域内的所有 suggestions 。

位于由 geohash 表示的区域内的 suggestions 也可以比其他 suggestion 更高，如下所示：

```bash
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "location": [ # ①
                    {
                        "lat": 43.6624803,
                        "lon": -79.3863353,
                        "precision": 2
                    },
                    {
                        "context": {
                            "lat": 43.6624803,
                            "lon": -79.3863353
                        },
                        "boost": 2
                    }
                 ]
            }
        }
    }
}
```

① 上下文查询过滤的 suggestions 落在由（43.662，-79.380）的 geohash 表示的地理位置（精度为2）下方的 suggestions ，并提升落在（43.6624803，-79.3863353）的 geohash 表示形式下的默认精度为6的 suggestions 乘以因子2。
_______________________


除了接受上下文值，上下文查询可以由多个上下文子句组成。 类别上下文子句支持以下参数：

参数   | 描述
------|------
`context` | 要过滤或提升 suggestion 的地理点对象或地理哈希字符串。 这是强制性的。
`boost` | 应该提高 suggestion 的分数的因子，通过将增强乘以 suggestion 权重来计算分数，默认为`1`
`precision	geohash` | 对查询地理点进行编码的精度。 这可以指定为距离值（`5m`，`10km`等），或作为原始 geohash 精度（`1..12`）。 默认为索引时间精度级别。
`neighbours` | 接受精度值数组，在该数组处应考虑相邻的地理散列。 精度值可以是距离值（`5m`，`10`km等）或原始 geohash 精度（`1..12`）。 默认为生成索引时间精度级别的邻居。
 