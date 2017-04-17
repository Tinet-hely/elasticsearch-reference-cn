# 在jvm.options中设置JVM堆大小

默认情况下,Elasticsearch告诉JVM使用堆的最小值和最大值的2GB。切换到生产时，保证Elasticsearch有足够的可用堆是非常重要的。

Elasticsearch将通过[jvm.options](./Configuring_system_settings.md#jvm-options)中的Xms（堆的最小值）与Xmx（堆的最大值）设置来分配堆的大小。

这个值依赖于服务器上可用的RAM数量，好的设置规则如下：

  * 堆的最小值（Xms）与堆的最大值（Xmx）设置成相同的。
  * Elasticsearch的可用堆越大，它能在内存中缓存的数据越多。但是需要注意堆越大在垃圾回收时造成的暂停会越长。
  * 设置Xmx不要大于物理内存的50%。用来确保有足够多的物理内存预留给操作系统缓存。
  * 不要设置Xmx超过JVM用来压缩对象指针的cutoff（compressed oops）；精确的cutoff可能不同，但接近于32GB。你可以通过在日志中查找一条类似于下面的这条信息来确定这个cutoff限制。
    ```bash
    heap size [1.9gb], compressed ordinary object pointers [true]
    ```
  * 最好尽量保持低于zero-based compressed oop的阈值；精确的cutoff可能不同，但大多数系统26GB是安全的，但是在某些系统可能多达30GB。你可以通过JVM的`XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode`参数来验证限制，并通过类似如下的行来确定：
    ```bash
    heap address: 0x000000011be00000, size: 27648 MB, zero based Compressed Oops
    ```
    如果是开启了zero-based compressed oop则
    ```bash
    heap address: 0x0000000118400000, size: 28672 MB, Compressed Oops with base: 0x00000001183ff000
    ```

下面演示了如何通过`jvm.options`文件来配置堆大小：

```yaml
-Xms2g #①
-Xmx2g #②
```

① 设置堆的最小值为2g。
___________________
② 设置堆的最大值为2g。

他们同样也能通过环境变量来设置。先需要在`jvm.options`文件中注释掉 `Xms`与`Xmx`设置，然后通过`ES_JAVA_OPTS`来设置：

```bash
ES_JAVA_OPTS="-Xms2g -Xmx2g" ./bin/elasticsearch #①
ES_JAVA_OPTS="-Xms4000m -Xmx4000m" ./bin/elasticsearch #②
```

① 设置堆的最小值与最大值为2GB。
___________________
② 设置堆的最小值与最大值为4000MB。

> 注意
>
> [Windows服务](../Installing_Elasticsearch/Install_Elasticsearch_on_Windows.md#windows-service)配置堆的大小与上面方式不同。初始值可以在安装Windows服务时配置，但是安装完之后也可以调整。查阅[Windows服务文档](../Installing_Elasticsearch/Install_Elasticsearch_on_Windows.md#windows-service)来获取更多信息。