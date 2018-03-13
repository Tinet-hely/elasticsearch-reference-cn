# Profiling Considerations

## 性能说明/Performance Notes

和任何探查器(Profiler)一样，Profile API为搜索执行引入了一个不可忽略的开销。检测低水平的方法调用的行为，如收集(collect),提前(advance)和下一个文档(next_doc)开销非常大，因为这些方法被称为紧环。因此，默认情况下不应在生产设置中启用探查(profiling)，不应与非探查(profile)的查询时间进行比较。探查(Profiling)只是一个诊断工具。

也有特殊Lucene优化被禁用的情况，因为它们是不适合探查(Profiling)。这可能会导致相比非探查形态，某些查询会报告较大的相关时间，但一般来说，和其他组件相比，在探查查询(profiled query.)中的应该不会有剧烈的影响。

## 局限性/Limitations

- 探查(Profiling)的分析统计目前不提供给`suggestions`，高亮(`highlighting`)， `dfs_query_then_fetch`。
- 聚集的减少阶段的探查(Profiling)目前不可用。
- 探查器(Profiler)仍然处于实验阶段。作为Lucene的检测部分，探查器(Profiler)从未被设计成以这种方式暴露出去，所以所有的结果都应该被看作提供详细诊断的最大努力。我们希望随着时间的不断推进改善这个模块。如果您发现明显错误的数字，奇怪的查询结构或其他错误，请提交报告。