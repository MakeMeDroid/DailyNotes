Lucene中的Packed Int通过紧凑的格式保存数值类型(long和int)。

需要注意的是：这个格式只保存无符号整数。存储负数也是可以的，但没有意义，因为所有负数long都得要求64位，packedint的优势也就荡然无存。



对于要保存的一系列数值，首先获取存储其最大值所需的位数(*bitsPerValue*)：

```java
Math.max(1, 64 - Long.numberOfLeadingZeros(maxNumber));
```

存储时每个数值只会占用bitsPerValue个位，所以如果数值的个数为N，则存储这些数组的总大小为`(N * bitsPerValue)`，暂且记为totalBits。

PackedInt提供了两种方式来存储这些bit：**long数组**和**byte数组**。

通常，写入数据用byte数组，而读取操作用long数组。



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

对上述接口的实现如下：

![PackedInt_codec](C:\Users\cyp\Desktop\Notes\Lucene\Utils\PackedInt_codec.png)

其中`BulkOperationPacked`为了提高读取数值的效率，对[1, 24]范围内的bitsPerValue通过子类`BulkOperationPacked<N>`重写了decode函数的实现。

`PackedInts.Format`定义了两种格式：**PACKED**和**PACKED_SINGLE_BLOCK**

其中PACKED为紧凑型存储格式，每个数值之间不会有填充字节；SingleBlock的版本中longBlockCount是固定的1，也就是一个block就是一个long，即64个bit。不是所有的bitsPerValue都能被64整除，所以最终的数据中会有部分bit充当padding的功能。



### DirectWriter和DirectReader

这两个类提供了直接向磁盘写入packedint形式的数据以及从磁盘数据通过packedint还原整型的功能。

这两个类只支持如下类型的bitsPerValue：

```java
int SUPPORTED_BITS_PER_VALUE[] = new int[] {
    1, 2, 4, 8, 12, 16, 20, 24, 28, 32, 40, 48, 56, 64
};
```

如果传入的bitsPerValue不在这个数组中，则会选择数组中比传入值大的最小值作为最终的bitsPerValue。



前面的内容说过，packedint的字节序列既可以用long[]存储，也可以用byte[]保存，DirectWriter采用byte[]的方式。因此Writer中需要两个数组来完成数据的写入：

```java
byte[] nextBlocks；
long[] nextValues；
```

其中，nextValues数组保存需要写入的整数，当nextValues写满或已经没有写入数据时，nextValues中的数据被编码为PackedInt的格式的字节序列存储到nextBlocks数组中，最后将nextBlocks中的数据写入磁盘。

因为PackedInt的数据需要尽量保持字节对齐，这样可以提高IO的效率，所以nextBlocks的长度得能保存整数倍的byteBlockCount，而nextValues中得保存整数倍的byteValueCount。因为byteValueCount个long需要byteBlockCount个字节进行存储，所以这两个倍数是相同的，假设这个倍数为N，则

```java
nextBlocks = new byte[N * byteBlockCount];
nextValues = new long[N * byteValueCount];
```

因此两个数组占用的空间大小为：
$$
N * (byteBlockCount + 8 * byteValueCount)
$$
在限定可用内存大小及总共需要写入的整数个数的条件下（DirectWriter用于存储写入数据及packedint类型的字节数组的空间大小为`PackedInts.DEFAULT_BUFFER_SIZE`，值为1024byte。），倍数N的计算过程如下：

```java
// BulkOperation.java(org.apache.lucene.util.packed.BulkOperation)

// valueCount为总共需要写入的整数数量，ramBudget为预计可用的内存大小
public final int computeIterations(int valueCount, int ramBudget) {
  // 因为(N *(byteBlockCount + 8 * byteValueCount)) <= ramBudget
  // 所以用如下表达式求出N的上限（iterations就代表N）
  final int iterations = ramBudget / (byteBlockCount() + 8 * byteValueCount());
  if (iterations == 0) {
    // 当预计内存太小时会走到这个分支，这时至少应该返回1
    return 1;
  } else if ((iterations - 1) * byteValueCount() >= valueCount) {
    // 到这一步，说明内存足够大，一次写入的整数个数超过了总共需要写入的个数，也即
    // N >= (valueCount / byteValueCount) + 1 (加1是因为需要对除法的结果向上取整保证空间大小足够)
    // 因此实际的N可以直接赋值为(valueCount / byteValueCount)的向上取整结果
    return (int) Math.ceil((double) valueCount / byteValueCount());
  } else {
    // 正常情况下直接返回第一步中的结果即可
    return iterations;
  }
}
```



DirectWriter中有一个细节需要注意，在`finish()`方法的结尾处会对磁盘上的数据追加三个空字节，这样做的好处如代码中的注释说是是为了提高IO的效率，这里提高的其实是DirectReader中对磁盘数据进行读取的IO效率。前面提到DirectWriter只支持特定的几种bitsPerValue，而这些数值对于8以下的都是按2的倍数递增，32以下是依次递增4，也就是半个字节，32之后的是依次递增一个字节。所以DirectReader在读取数据时根据bitsPerValue采用如下方式：

[1, 2, 4, 8]每次读取一个byte；

[12, 16]每次读取一个short；

[20, 24, 28, 32]每次读取一个int；

[40, 48, 56, 64]每次读取一个long

对于20和24，必须多读出一个字节的数据才能组成一个int；对于40，需要多读出3个，48多读2个，56多读1个。所以只要在末尾添加最多3个空字节就能满足所有情况下的读取需求了。



DirectReader为每个bitsPerValue值创建一个`LongValues`的子类`DirectPackedReader[BPV]`对数据进行读取。



### DirectPackedReader、DirectPacked64SingleBlockReader

这两个类继承PackedInts.ReaderImpl，两个类读取的数据分别对应**PACKED**和**PACKED_SINGLE_BLOCK**格式。

两者都是直接通过index计算数据所在的位置，并直接从磁盘中读取相应数据的实现方式。因为每个数值都会从磁盘中读数据，所以这两个类的执行效率并不高。



### Packed64、Packed64SingleBlock

两个类都提供了对数据进行读写的操作，并且读写都有单值和批量两种方式。

Packed64先把数据存储于long数组中，然后按packedInt的格式对数据进行读取，批量读写用到了BulkOperation类的功能。

Packed64SingleBlock的功能和Packed64类似，区别是其解析数据是按照PACKED_SINGLE_BLOCK的方式进行的。



### PackedWriter

接口`PackedInts.Writer`唯一的实现类，其写入数据的流程和`DirectWriter`基本一样，这个类中多了个写入文件头的函数，文件头中包含bitsPerValue和valueCount等信息。



### BlockPackedWriter、BlockPackedReader、BlockPackedReaderIterator

BlockPackedWriter也是采用packedint的格式存储数字，存储方式如下：

1. 将需要存储的数据分为一系列大小为blockSize的块（blockSize为2的整数次幂）
2. 每个block的数据存储之前先确定数据中的最大值和最小值以及最大值与最小值之间的差值
3. 在磁盘文件中先存储表示差值所需的bit个数以及用zigzag编码的最小值
4. 最后存储原始数据与最小值的差值（如果所有数据相同，则这一步可以省略，因为这时候差值所需bit数为0）。



BlockPackedReader把每个block解析为一个最小值和一个PackedInts.Reader类型的对象，并把它们保存到各自的数组中，然后根据index从Reader中读取数据。



BlockPackedReaderIterator提供了一个迭代器用于读取BlockPackedWriter写入的数据，可以每次读取一个，或者是一次读取多个。这个迭代器没有实现Java中的Iterator接口，所以没有相应的hasNext()接口，而是提供了skip(long)，next(), next(count)等接口。



### DirectMonotonicWriter、DirectMonotonicReader

DirectMonotonicWriter用于保存单调递增的整数数列。

写入的数据被划分为一系列block，因此对象创建时需要传入blockShift参数，每个block的包含的整数个数为
$$
2^{blockShift}
$$

存储n个数据就需要`((n - 1) >>> blockShift) + 1`个block。

因为每个block中的数据都是单调递增的，所以这里采用了保存数据增量的方式存储数据，以有效压缩数据的存储大小。过程如下：

1. 计算block中数据的平均增量。用最后一个数据与第一个数据的差值除以数据之间间隔数（也就是数据总数减1）。比如

   ```
   对于数组[2, 5, 6, 10]
   数据个数为4，数据之间的间隔数为3（逗号的数量）
   那么这个数组的平均增量为(10 - 2) / 3 = 2.667
   ```

   这个平均增量记为`AvgIncrement`。

2. 计算每个数据相对于平均增量计算出的期望值的偏移量

   ```
   还是以上面的例子为例，计算得出AvgIncrement为2.667，这样数组中第i个位置的元素的期望值为(AvgIncrement * i)的取整，i从0开始；
   因此数组用AvgIncrement计算出的期望值为：[0, 2, 5, 8];
   然后数组中每个元素相对于期望值的偏移量为：[2, 3, 1, 2]
   ```

3. 找出步骤2中的最小偏移量。在上面的例子中为第三个偏移量：1。

4. 计算出步骤2中每个偏移量减去最小偏移量的值，并找出最终的最大值（记为maxDelta）：

   ```
   在上面的例子中，最小的偏移量为1，因此，减去最小偏移量之后的结果为：
   [2, 3, 1, 2] - 1 = [1, 2, 0, 1]
   最终结果中，最大值maxDelta为2
   ```

5. 计算保存maxDelta所需的最大位数。如果步骤4中的maxDelta为0，说明block中的数据为等差数列，只需要在元数据文件中相应字段中存储0即可；如果maxDelta不为0，则需要在索引数据中存储步骤4的计算结果(也就是例子中的[1, 2 ,0, 1]数组)。



整个过程设计到两个文件，一个用于保存元数据信息，另一个用于保存数据信息。元数据信息包括步骤3中获取的最小值、步骤1中的AvgIncrement、当前block的数据在数据文件中的偏移量以及存储maxDelta所需的位数。真实数据则通过[DirectWriter](#DirectWriter和DirectReader)写入数据文件。



DirectMonotonicReader会先读取元数据文件的信息到内存，然后根据这些信息还原数据文件中的数据。除了通过index读取数据外，此类还提供了通过二分法在指定索引范围内查找某个值的方法。能进行二分查找的关键是对于一个给定的index，其可以根据元数据信息确定一个范围，因为元数据信息中记录了表示maxDelta的位数，所以delta的变化范围是[0, maxDelta]，通过delta也就可以确定真实数据的范围。



### MonotonicBlockPackedWriter、MonotonicBlockPackedReader

这两个类处理的也是单调递增的整数数列，写入的流程和DirectMonotonicWriter一模一样。

想吐槽的一点是DirectMonotonicWriter写入流程中的步骤2和3在这个类中写在了一个for循环中，理解起来很是晦涩，不用公式推导一下真不知道到底在计算啥。接下来我们站在写这个代码的人的角度来考虑这段代码应该怎么写：

首先，每个值相对于avg的偏移量是`(values[k] - avg * k)`,在所有这些偏移量中，肯定存在某个索引m使得其对应的偏移量`(values[m] - avg * m)`是其中的最小值，记为min；所以对于每个索引`i`，最终要保存的数值是`(values[i] - avg * i - min)`，并且这个值肯定是大于等于0的。因此代码中先假设这个min是`(values[0] - avg * 0)`，也即values[0]，然后根据这个最小值计算索引`i`上的期望值expected[i]为`(min + avg*i)`，如果这个值大于values[i]，说明这个min不是最小值，而应该以`(values[i] - avg * i)`取代原来的min，而`(values[i] - avg*i) <=> values[i] - (min + avg*i) + min <=> min - (expected[i] - values[i])`。这就是代码中计算min的表达式的由来，余下的部分和DirectMonotonicWriter中的处理就是一样的了。

需要注意的是这个类保存数据只用到了一个文件，也就是元数据信息和数据是存在一起的，这种方式显然没有把元数据信息单独存储的方式灵活，因为元数据信息不能一次性读取到内存，所以二分查询也就难以实现。这就是为啥MonotonicBlockPackedReader只提供了按索引读取数据的方法。



### PackedLongValues、DeltaPackedLongValues、MonotonicLongValues

（待续）



### GrowableWriter

此类实现了`PackedInts.Mutable`接口，提供了读写整型数据的功能，内部存储用packedInt格式。它有一个初始bitsPerValue，当写入的值需要的位数大于初始的bitsPerValue时，会重新选择合适的bitsPerValue并重新构建之前的数据，这就是`Growable`的含义。



### PackedDataInput、PackedDataOutput

（待续）

