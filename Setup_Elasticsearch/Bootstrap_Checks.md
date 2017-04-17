# 启动前检查

## 启动检查

总的来说，我们有很多用户遇到一些意想不到的问题都是因为他们没有配置[重要设置](./Important_Elasticsearch_configuration.md)。在Elasticsearch以前的版本中，其中的一些配置错误被记录为警告信息。可以理解的是，用户有时会忽略这些日志消息。为了确保这些设置受到应有的重视，Elasticsearch在启动之前会做一些检查。

这些启动查会检测Elasticsearch和系统的各种设置，并比较与那些为值是否是Elasticsearch的操作安全值。如果Elasticsearch处于开发模式，任何启动检失败被记录为Elasticsearch警告日志。如果Elasticsearch是在生产模式下，任何启动检查失败会导致Elasticsearch拒绝启动。

有一些引导检查总是执行是为了防止Elasticsearch使用不兼容的设置。这些检查是单独记录。

## <span id="dev-vs-prod">开发模式vs生产模式</span>

默认情况下，Elasticsearch为[HTTP模块](../Modules/HTTP.md)和[传输（内部）模块](../Modules/Transport.md)的绑定在`localhost`上通信。这是对于下载与试玩Elasticsearch、以及日常开发都是非常好的，但是对生产系统无用的。为了形成一个群集，Elasticsearch实例各节点之前传输通信必须绑定外部的网络接口。因此，我们判定一个Elasticsearch实例是否已开发模式运行就是看它是否绑定了外部网络接口（默认是开发模式运行）。注意，HTTP和传输模块可以通过`http.host`与`transport.host`独立配置；这可以用来配置一个HTTP外部可访问的单一实例来避免触发生产模式的一些检查。