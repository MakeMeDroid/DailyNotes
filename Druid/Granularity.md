Granlarity用于定义数据或者segment的时间粒度。

粒度本质上是一个分组概念，而时间粒度也即对时间进行分组。`Granulariy`是所有时间粒度的一个抽象，其包含了一个具体时间粒度可能执行的操作。具体的时间粒度主要有三种：

- `AllGranularity`
- `NoneGranularity`
- `PeriodGranularity`
- `DurationGranularity`



## JSON序列化

Druid的**processing**模块中包含了`JacksonModule`模块，其中定义了系统执行过程中需要用JSON进行编解码的类的序列化与反序列化方法（通过Jackson的module接口）。其中就包含了`GranularityModule`，它为上述四个Granularity建立subType，这样就能根据JSON的type内容灵活的解析为具体的粒度。



## Segment粒度

任何一个Segment的粒度只能是`GranularityType`中定义的粒度（排除`ALL`和`NONE`）:

1. `SECOND`
2. `FIVE_SECOND`
3. `TEN_SECOND`
4. `FIFTEEN_SECOND`
5. `THIRTY_SECOND`
6. `MINUTE`
7. `FIVE_MINUTE`
8. `TEN_MINUTE`
9. `FIFTEEN_MINUTE`
10. `THIRTY_MINUTE`
11. `HOUR`
12. `TWO_HOUR`
13. `SIX_HOUR`
14. `DAY`
15. `WEEK`
16. `MONTH`
17. `QUARTER`
18. `YEAR`

通常用户会在入库任务的配置中指定Segment的粒度，这个配置值对于Druid来说只是一个粒度的上界（也即最终的Segment粒度不会比这个配置粒度大）。此外入库任务配置中还有一个数据粒度配置项，这个粒度用于把每行数据中的时间字段截断为粒度对应的时间精度，这个粒度决定了最终Segment粒度的下界。所有最终Segment的粒度只会是上述`GranularityType`定义的粒度列表中位于[dataGranularity, segmentGranularity]范围内的某个粒度。

根据SegmentID的分配规则，数据粒度与segment粒度必须满足如下条件：

**每个segment粒度区间都恰好包含整数个数据粒度区间**

这样，每个数据粒度区间都只会被一个segment粒度区间包含。如果不满足这个条件，新SegmentID的创建可能会以失败告终。换句话说，就是合法的数据粒度应该保证如下规则：

假设某个时间点的数据基于数据粒度计算出来的时间区间为row_interval，那么必须存在至少一个segment粒度，使得任意时间点的row_interval都只能与一个segment粒度对应的区间有交集。

逻辑上，一个DataSource的segment粒度与数据粒度都是可以修改的，但修改时应该遵守如下规则：

**新的数据粒度不能超过从创建DataSource以来使用过的最小的segment粒度**