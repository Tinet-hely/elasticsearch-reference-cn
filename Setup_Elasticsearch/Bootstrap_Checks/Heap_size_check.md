# 堆大小检查

如果JVM采用不同的初始与最大的堆（heap）值启动，它容易在系统使用时堆大小发生变化时造成JVM暂停。为了避免这些调整的停顿，最好设置JVM的初始堆大小等于最大堆大小。此外，如果[bootstrap.memory_lock](../Important_Elasticsearch_configuration.md#bootstrap.memory_lock)被启用，在启动时JVM将锁定堆的初始大小。如果初始堆大小不等于最大堆大小，后面将不会调整大小因为所有JVM堆都被在内存中锁定。要通过堆大小检查，则必须配置[堆大小](../Important_System_Configuration/Set_JVM_heap_size_via_jvm.options.md)。