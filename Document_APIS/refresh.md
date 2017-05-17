# ?refresh （冲刷）

[Index](./Index_API.md)、[Update](./Update_API.md)、[Delete](./Delete_API.md)和[Bulk](./Bulk_API.md)API支持设置`refresh`以控制此请求所做的更改对搜索可见。这些是允许的值：

空字符串或`true`

&emsp;&emsp;在操作发生后立即刷新相关的主要和副本分片（不是整个索引），以便更新的文档立即显示在搜索结果中。只有仔细思考和验证才能从索引和搜索的角度出发，不会导致性能不佳。

`wait_for`

&emsp;&emsp;等待请求所做的更改在返回之前通过冲刷显示。这不会强制立即刷新，而是等待刷新发生。 Elasticsearch会自动每隔`index.refresh_interval`刷新已经更改的分片，默认为`1`秒。该设置是[动态](../Index_Modules.md#dynamic-index-settings)的。调用[Refresh](../Indices_APIs/Refresh.md) API或将任何支持该API的`refresh`设置为`true`也将导致刷新，从而导致已经运行的请求与`refresh=wait_for`返回。

假（默认）

&emsp;&emsp;不要刷新相关的动作。在请求返回后，此请求所做的更改将在某个时刻显示。

## 选择哪个设置来使用

除非你有一个很好的理由等待修改变得可见，总是使用`refresh=false`，或者，因为这是默认值，只需将刷新参数退出URL。那是最简单和最快的选择。

如果你一定要让所做的修改请求同步可见，那么您必须在对Elasticsearch（true）进行更多的负载与更长的等待响应（`wait_for`）之间进行选择。这里有几点应该告诉这个决定：

- 与设置为`true`相比，`wait_for`能让索引做更多的变更工作，在这种情况下，每隔`index.refresh_interval`索引的修改只才会保存。
- `true`将构造较小的有效的索引（微小段），以后必须将其合并到更有效的索引构造（较大的段）中。这意味着设置为`true`时，索引将花费时间在创建微小段上面，在搜索时从微小段进行搜索，并在合并时来制作较大段。
- 不要在一行中启动多个`refresh=wait_for`请求。而是通过一个Bulk请求来使用`refresh=wait_for`，Elasticsearch将并行执行它们，并且只有当它们全部完成时才返回。
- 如果刷新间隔设置为`-1`，则禁用了自动刷新，则`refresh=wait_for`的请求将无限期地等待，直到某些其它操作导致刷新。相反，将`index.refresh_interval`设置为小于默认值譬如`200ms`，`refresh=wait_for`更快地恢复，但仍会生成低效的段。
- `refresh=wait_for`仅影响其所在的请求，但是，通过强制立即刷新，`refresh=true`将影响其他正在进行的请求。一般来说，如果你有一个运行的系统，你不想打扰，那么`refresh=wait_for`是一个较小的修改。

## `refresh=wait_for`能强制刷新

如果一个`refresh=wait_for`请求进来，当已经有`index.max_refresh_listeners`（默认为`1000`）请求在等待该分片上的刷新时，那么该请求的行为就好像`refresh`设置为`true`：它将强制刷新。这保证了当`refresh=wait_for`请求返回其更改对于搜索是可见的时候，同时防止阻止请求的未检查的资源使用。如果一个请求被强制刷新，因为它超出监听器插槽，则其响应将包含`"forced_refresh":true`。

Bulk请求只占用接触的每个分片上的一个`slot`，无论他们修改分片多少次。

## 示例

这些将创建一个文档并立即刷新索引，使其可见：

```js
PUT /test/test/1?refresh
{"test": "test"}
PUT /test/test/2?refresh=true
{"test": "test"}
```

这些将创建一个文档，而不做任何使其可以搜索的事情：

```js
PUT /test/test/3
{"test": "test"}
PUT /test/test/4?refresh=false
{"test": "test"}
```

这将创建一个文档并等待它成为搜索可见：

```js
PUT /test/test/4?refresh=wait_for
{"test": "test"}
```