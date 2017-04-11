# Lucence表达式语言

Lucene表达式将编译成一个javascript表达式的字节码。它们是专为高性能的自定义评分和排序功能设计，默认情况下启用内联和存储脚本。


## 性能

通过定制Lucene代码， 表达式被设计得具有不错竞争力的性能。这种表现是由于具有每个文档的低开销，相对于其他的脚本引擎：Lucene表达式在“前期”做了更多的事情。

这使操作变得非常快，甚至比你写一个`native`脚本。


## 语法

表达式支持javascript语法的子集：一个单一的表达。

更多操作符与函数的可用信息，请参阅[apache lucene表达式模块文档](http://lucene.apache.org/core/6_0_0/expressions/index.html?org/apache/lucene/expressions/js/package-summary.html)。

表达式中变量的脚本可以访问：

- 文档字段，如：`doc['myfield'].value`
- 变量和方法的支持，如：`doc['myfield'].empty`
- 参数传递到脚本，如：`mymodifier`
- 当前文档的分数，`_score`（仅适用于在使用`script_score`时）

您可以`script_score`、`script_fields`、脚本排序、数字型字段的脚本聚合等场景中使用表达式脚本，简单地设置下表达式语言参数。


## 数字类型字段API

| 表达式                        |描述
| :----------------------------|:-------------------------------
|doc['field_name'].value       |该字段的值，double类型
|doc['field_name'].empty       |该字段是否有值，返回true/false
|doc['field_name'].length      |该字段的值的本文长度
|doc['field_name'].min()       |本文档中此字段的最小值
|doc['field_name'].max()       |本文档中此字段的最大值
|doc['field_name'].median()    |本文档中此字段的中间值
|doc['field_name'].avg()       |本文档中此字段的平均值
|doc['field_name'].sum()       |本文档中此字段的总和

如果文档中缺失这个字段，默认将按值为`0`来处理。你也可以用其它默认值来处理，譬如通过表达式：`doc['myfield'].empty ? 100 : doc['myfield'].value`

如果文档中这个字段有多个值，默认情况下最小的值被返回。您可以选择不同的值来代替，譬如返回这个字段的和：`doc['myfield'].sum()`

如果文档中缺失这个字段，默认值将被视为`0`。

布尔类型字段如果被拿来当数字字段进行计算时，`true`将被映射为`1`，`false`将被映射为`0`。例如：`doc['on_sale'].value ? doc['price'].value * 0.5 : doc['price'].value`


## 日期类型字段API

日期类型的字段奖被按照自1970年1月1日起的毫秒数，完全支持上面的数字类型字段API。以及下面一些特定的日期字段运算：

| 表达式                                     |描述
| :-----------------------------------------|:-------------------------------
|doc['field_name'].date.centuryOfEra        |当前世纪（1-2920000）
|doc['field_name'].date.dayOfMonth          |当前月中的第几天（1-31），例如：1表示在当月中的第一天。
|doc['field_name'].date.dayOfWeek           |当前周中的第几天（1-7）天，例如：1表示星期一。
|doc['field_name'].date.dayOfYear           |当前年中的第几天，例如：1表示1月1日。
|doc['field_name'].date.era                 |时代，0表示公元前，1表示公元。
|doc['field_name'].date.hourOfDay           |小时（0-23）。
|doc['field_name'].date.millisOfDay         |当前天中的第几毫秒数（0-86399999。
|doc['field_name'].date.millisOfSecond      |当前秒钟的第几毫秒（0-999）。
|doc['field_name'].date.minuteOfDay         |当前天中的第几分钟（0-1439）。
|doc['field_name'].date.minuteOfHour        |当前小时中的第几分钟（0-59）。
|doc['field_name'].date.monthOfYear         |当前年中的第几个月（1-12），例如：1表示一月。
|doc['field_name'].date.secondOfDay         |当前天中的第几秒（0-86399）。
|doc['field_name'].date.secondOfMinute      |当前分中的第几秒（0-59）。
|doc['field_name'].date.year                |年（-292000000 - 2.92亿）。
|doc['field_name'].date.yearOfCentury       |当前世纪中的第几年（1-100）。
|doc['field_name'].date.yearOfEra           |当前时代中的第几年（1-292000000）。

下面的例子展示了如何计算`date1`与`date0`字段相差几年：

`doc['date1'].date.year - doc['date0'].date.year`


## 地理点（geo）类型字段API

| 表达式                                     |描述
| :-----------------------------------------|:-------------------------------
|doc['field_name'].empty       |该字段是否有值，返回true/false
|doc['field_name'].lat         |地理点的纬度。
|doc['field_name'].lon         |地理点的经度。

下面的例子展示了计算`field_name`字段值到华盛顿有多少公里：

`haversin(38.9072, 77.0369, doc['field_name'].lat, doc['field_name'].lon)`

在这个例子中的坐标可能被作为参数传递给脚本，例如: 用户的当前地理位置。


## 限制

相对于其他脚本语言的一些限制：

- 只有数字、布尔、日期和地理点类型的字段可以被访问；
- 存储字段（Stored fields）不可用
