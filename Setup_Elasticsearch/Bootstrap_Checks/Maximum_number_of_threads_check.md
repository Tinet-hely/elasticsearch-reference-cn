# 最大线程数检查

Elasticsearch执行请求时将请求分解成多个阶段,在不同阶段使用不同的线程池执行。elasticsearch有执行不同任务的[线程池执行器](../../Modules/Thread_pool.md)。因此Elasticsearch需要具备创建很多线程的能力。最大线程数检查是确保Elasticsearch进程有创建足够多线程来正常使用的权利。此检查只针对Linux系统。如果你是在Linux上运行，要通过此检查你必须让Elasticsearch进程至少有创建`2048`个线程的能力。可以通过`/etc/security/limits.conf`使用`nproc`来配置（注意：你可能还需要增加`root`用户的相关限制）。