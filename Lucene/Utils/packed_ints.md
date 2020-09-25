Lucene中的Packed Int通过紧凑的格式保存数值类型(long和int)。

需要注意的是：这个格式只保存无符号整数。存储负数也是可以的，但没有意义，因为所有负数long都得要求64位，packedint的优势也就荡然无存。



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



## byte数组存储

byte数组的存储原理和long相同，只不过一个block需要的byte数记为byteBlockCount，而一个block中能保存多少数字记为byteValueCount，它们有如下关系：
$$
\frac{8}{bitsPerValue} = \frac{byteValueCount}{byteBlockCount}
$$
所以byteBlockCount和byteValueCount分别是8和bitsPerValue化简之后的值（除以最大公约数）。



## 接口

Lucene中packedint对数值进行编码的接口为：

```java
// 把一系列数值通过packed int的方式存储至byte数组或long数组
public static interface org.apache.lucene.util.packed.PackedInts.Encoder {
    int longBlockCount();
    int longValueCount();
    int byteBlockCount();
    int byteValueCount();
    void encode(long[] values, int valuesOffset, long[] blocks, int blocksOffset, int iterations);
    void encode(long[] values, int valuesOffset, byte[] blocks, int blocksOffset, int iterations);
    void encode(int[] values, int valuesOffset, long[] blocks, int blocksOffset, int iterations);
    void encode(int[] values, int valuesOffset, byte[] blocks, int blocksOffset, int iterations);
}
```

参数说明：

`values`参数为需要写入的数据值，类型可以为long或者int，`valuesOffset`为values中数据开始的偏移量；

`blocks`用于存储通过packedint对values编码后的结果，`blocksOffset`为blocks中开始存储数据的偏移量；

`iterations`为写入的block的数量，因为每个block可以存储`longValueCount`/`byteValueCount`个数值，所以这个参数表示需要从`values`中读取的数值个数为`(interations * longValueCount)`或者`(interations * byteValueCount)`。



解码接口：

```java
public static interface org.apache.lucene.util.packed.PackedInts.Decoder {
    int longBlockCount();
    int longValueCount();
    int byteBlockCount();
    int byteValueCount();
    void decode(long[] blocks, int blocksOffset, long[] values, int valuesOffset, int iterations);
    void decode(byte[] blocks, int blocksOffset, long[] values, int valuesOffset, int iterations);
    void decode(long[] blocks, int blocksOffset, int[] values, int valuesOffset, int iterations);
    void decode(byte[] blocks, int blocksOffset, int[] values, int valuesOffset, int iterations);
}
```

Decoder接口中的参数作用与Encoder中的一致，只不过Decoder是从`blocks`中解析出数值序列，并存储于`values`中。



### 实现

对上述