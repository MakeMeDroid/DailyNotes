Lucene中的Packed Int通过紧凑的格式保存数值类型(long和int)。

需要注意的是：

1. 这个格式只保存无符号整数；
2. 



对于要保存的一系列数值，首先获取存储其最大值所需的位数(*bitsPerValue*)：

```java
Math.max(1, 64 - Long.numberOfLeadingZeros(maxNumber));
```

存储时每个数值只会占用bitsPerValue个位，所以如果数值的个数为N，则存储这些数组的总大小为`(N * bitsPerValue)`，暂且记为totalBits。

PackedInt提供了两种方式来存储这些bit：**long数组**和**byte数组**。

## long数组存储

一个long可以存储64位，但bitsPerValue不一定能整除64，所以需要选择某个数量的long，使其正好能存储整数个bitsPerValue，long的数量在实现中记为`longBlockCount`，而这么多个long能存储的bitsPerValue的数量记为longValueCount，它们有如下关系：
$$
64 * longBlockCount = bitsPerValue * longValueCount
$$
或
$$
\frac{64}{bitsPerValue} = \frac{longValueCount}{longBlockCount}
$$
从公式中可以看出，满足条件的longValueCount和longBlockCount有很多解，我们只使用这些解中的最小值。因为64的2的次方，所以longBlockCount的值可以通过如下方式求解：

1. 如果bitsPerValue为奇数，则longBlockCount的值为bitsPerValue；
2. 如果bitsPerValue为偶数，则持续对其除以2，直到其值为奇数（最小为1），这个奇数即为longBlockCount的值。

