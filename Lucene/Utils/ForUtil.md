Lucene的`org.apache.lucene.codecs.lucene84`包下定义了一个类`ForUtil`，其提供的功能和PackedInts相同，就是只通过最少的位数来表示每个整数，并将其存储与Long数组中。初次查看其计算过程时颇为费解，只不过看懂之后就觉得很简单了。



类中两个主要函数为：

```java
void encode(long[] longs, int bitsPerValue, DataOutput out)
void decode(int bitsPerValue, DataInput in, long[] longs)
```

encode负责把数组`longs`中的数字压缩存储到long型数组中，并写入DataOutput；

decode负责从DataInput中读取压缩数据，并解压到数组`longs`中。



首先需要明白的是，之所以类中的计算过程看上去繁琐，是因为这种实现的计算过程可以利用向量化计算的特性（SIMD指令集），提高计算的处理速度，具体可以参考[1]。



encode的计算过程就是一个不同数字的位合并的过程，俗话说，"talk is cheap, show me your pictures"，所有接下来咱们用图形来说明encode的过程。

首先，encode每次只会处理固定的128个数字



参考：

[1] https://fulmicoton.com/posts/bitpacking/

