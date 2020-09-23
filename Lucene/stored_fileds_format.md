# Stored Fileds Format

用于存储文档内容，采用两种可选的压缩算法：**LZ4**和**DEFLATE**，前者注重速度（BEST_SPEED），后者注重压缩率（BEST_COMPRESSION）。

内容分三个文件存储：

1. 数据文件（.fdt）

   存储按block压缩的文档数据。

2. 索引文件（.fdx）

   存储两个数组，一个指示每个block的首个doc ID，另一个存储block的文件偏移量

3. 元数据文件（.fdm）

   保存与索引文件中数组相关的元数据信息。



`StoredFieldsWriter`接口用于存储文档的字段数据，其工作流程为：

```java
for(Document doc : documents)
    startDocument()  //表示一个新文档的开始
    for(Field field : doc.fields)
        writeField(fieldInfo, indexableField) //写入每个字段的信息
    finishDocument() //表示一个文档写入完成

finish(FieldInfos, numDocs) //文档写入完毕
close() //关闭相关的资源
```



## fdt

### 文件头

| 名称                | 内容                                                      | 大小(bytes) |
| ------------------- | --------------------------------------------------------- | ----------- |
| codec_magic         | 0x3fd76c17                                                | 4           |
| codecLength         | length(codec) = 28                                        | 1（VInt）   |
| codec               | Lucene87StoredFieldsFastData/Lucene87StoredFieldsHighData | 28          |
| version             | 3                                                         | 4           |
| segmentID           |                                                           | 16          |
| segmentSuffixLength |                                                           | 1           |
| segmentSuffix       |                                                           | [0, 255]    |



### 数据块（Chunck）

文件头之后是Document的数据信息，其被组织为Chunck进行存储，每个Chunck包含若干个Document，Document的信息会先被缓存到内存中，如果缓存中保存的Document个数（默认chunck中最大文档数为512）或者缓存的总字节数超过设置的阈值，则会将当前缓存的文档信息打包为一个Chunck存储到fdt文件中。

其中每个Chunck的结构为

| 名称                 | 描述                                                         | 类型   |
| -------------------- | ------------------------------------------------------------ | ------ |
| docBase              | 块中第一个文档的ID（实际就是当前块之前已写入的文档数）       | VInt   |
| numDocs \| slicedBit | numDocs是当前块中的文档数，其向左移了一位用于存储slicedBit，slicedBit标识此块是否被切割 | VInt   |
| numStoredFields      | 长度等于numDocs的数组，其保存每个文档中的字段数              | VInt[] |
| lengths              | 长度等于numDocs的数组，其保存每个文档的字节大小（未压缩的大小） | VInt[] |
| data                 | Chunck的数据内容（压缩后）                                   | byte[] |

当缓存的数据量大于两倍设置的chuncksize时，数据会被切片（slice），然后对每个slice进行压缩并存储，所以表格中的slicedBit就是用于指示块中的数据是否被划分为slice。

其中data解压后的格式为：

| 字段        | 描述                             | 类型  |
| ----------- | -------------------------------- | ----- |
| infoAndBits | fieldNumber << TYPE_BITS \| bits | VLong |
| fieldValue  | 根据字段的类型存储对应的数据     |       |

其中字段类型可以为：

```java
static final int         STRING = 0x00;
static final int       BYTE_ARR = 0x01;
static final int    NUMERIC_INT = 0x02;
static final int  NUMERIC_FLOAT = 0x03;
static final int   NUMERIC_LONG = 0x04;
static final int NUMERIC_DOUBLE = 0x05;
```

fieldValue的存储格式：

**ByteArray**

| 字段   | 描述         | 类型   |
| ------ | ------------ | ------ |
| length | 字节数组长度 | VInt   |
| bytes  | 字节数组内容 | Byte[] |

**String**

| 字段   | 描述                      | 类型   |
| ------ | ------------------------- | ------ |
| length | 字符串长度（UTF-8）       | VInt   |
| bytes  | 字符串的字节表示（UTF-8） | Byte[] |

**Byte**/**Short**/**Integer**

| 字段   | 描述 | 类型 |
| ------ | ---- | ---- |
| number | 数值 | VInt |

**Long**

| 字段      | 描述                           | 类型  |
| --------- | ------------------------------ | ----- |
| header    | 如果是时间戳字段，标记时间粒度 | Byte  |
| upperBits | long的zigzag编码字节           | VLong |

Long字段写入前会先做timestamp的判断（目的是做压缩处理），判断的结果用header的前两个位表示：

- 00 - 表示无需压缩
- 01 - 表示值为1000的倍数（也就是说可以看作是粒度为秒的时间戳）
- 10 - 表示值为3600000的倍数（粒度为小时的时间戳）
- 11 - 表示值为86400000的倍数（粒度为天的时间戳）

header剩下的6位用于存储long的部分数据，所以第三位表示之后是否有有效数据（和VInt原理一样），然后剩下的5位存储long数据的低5位，long的其它位通过VLong的形式存储于上述表格中的upperBits字段中。

参考：*org.apache.lucene.codecs.compressing.CompressingStoredFieldsWriter.writeTLong(DataOutput out, long l)*

**Float**

| 字段   | 描述                 | 类型   |
| ------ | -------------------- | ------ |
| header | 决定存储字节数的标记 | Byte   |
| bytes  | 存储float数据的字节  | Byte[] |

参考：*org.apache.lucene.codecs.compressing.CompressingStoredFieldsWriter.writeZFloat(DataOutput out, float f)*

**Double**

| 字段   | 描述                 | 类型   |
| ------ | -------------------- | ------ |
| header | 决定存储字节数的标记 | Byte   |
| bytes  | 存储Double数据的字节 | Byte[] |

参考：*org.apache.lucene.codecs.compressing.CompressingStoredFieldsWriter.writeZDouble(DataOutput out, double d)*



### 文件尾

| 名称         | 内容                             | 大小(bytes) |
| ------------ | -------------------------------- | ----------- |
| footer_magic | ~CODEC_MAGIC                     | 4           |
| 固定常量     | 0                                | 4           |
| CRC          | 文件中除文件尾之外的内容的校验码 | 8           |



## fdx

fdx文件保存查找fdt文件中数据的索引信息。文档写入过程中并没有直接写fdx文件，而是先写入两个临时文件（非完整文件名）：

- Lucene85FieldsIndex-doc_ids

  保存fdt中每个chunck中的文档数

- Lucene85FieldsIndexfile_pointers

  保存fdt中每个chunck相对于前一个chunck的文件偏移量（也就是前一个chunck占用文件空间的大小）

这两个临时文件除了数据内容外，拥有与fdt格式相同的文件头和文件尾，这里不再赘述。当一个segment写入完毕后，fdx的数据内容才会根据这两个临时文件的内容生成并写入fdx文件中。



以下为fdx的文件格式

### 文件头

| 名称                | 内容                   | 大小(bytes) |
| ------------------- | ---------------------- | ----------- |
| codec_magic         | 0x3fd76c17             | 4           |
| codecLength         | length(codec) = 22     | 1（VInt）   |
| codec               | Lucene85FieldsIndexIdx | 22          |
| version             | 0                      | 4           |
| segmentID           |                        | 16          |
| segmentSuffixLength |                        | 1           |
| segmentSuffix       |                        | [0, 255]    |



### 索引数据

| 字段 | 描述 | 类型 |
| ---- | ---- | ---- |
|      |      |      |
|      |      |      |
|      |      |      |

索引数据中保存两种数据：chunck中的文档数和chunck在fdt文件中的起始偏移量。

两种数据是分开存储的，索引文件中先存储了文档数信息，然后存储偏移量信息。因为这两种数据都是long型数据，并且都是单调递增的数列，所以在索引文件中都采用相同的存储格式。

数据被组织为block，每个block的大小为
$$
2^{blockShift}
$$

其中blockShift为fdm文件中记录的一个元数据信息，是写入数据前用户设置的一个参数。这样存储n个数据就需要`((n - 1) >>> blockShift) + 1`个block。

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
   然后数组中每个元素现对于期望值的偏移量为：[2, 3, 1, 2]
   ```

3. 找出步骤2中的最小偏移量。在上面的例子中为第三个偏移量：1。

4. 计算出计算出步骤2中每个偏移量减去最小偏移量的值，并找出最终的最大值（记为maxDelta）：

   ```
   在上面的例子中，最小的偏移量为1，因此，减去最小偏移量之后的结果为：
   [2, 3, 1, 2] - 1 = [1, 2, 0, 1]
   最终结果中，最大值maxDelta为2
   ```

5. 计算保存maxDelta数组所需的最大位数。如果步骤4中的maxDelta为0，说明block中的数据为等差数列，只需要在元数据文件中相应字段中存储0即可；如果maxDelta不为0，则需要在索引数据中存储步骤4的计算结果(也就是例子中的[1, 2 ,0, 1]数组)。



### 文件尾

| 名称         | 内容                             | 大小(bytes) |
| ------------ | -------------------------------- | ----------- |
| footer_magic | ~CODEC_MAGIC                     | 4           |
| 固定常量     | 0                                | 4           |
| CRC          | 文件中除文件尾之外的内容的校验码 | 8           |



## fdm

### 文件头

| 名称                | 内容                    | 大小(bytes)    |
| ------------------- | ----------------------- | -------------- |
| codec_magic         | 0x3fd76c17              | 4              |
| codecLength         | length(codec) = 23      | 1（VInt）      |
| codec               | Lucene85FieldsIndexMeta | 23             |
| version             | 3                       | 4              |
| segmentID           |                         | 16             |
| segmentSuffixLength |                         | 1              |
| segmentSuffix       |                         | [0, 255]       |
| chunkSize           |                         | [1, 5]（VInt） |
| PackedIntsVersion   | 2                       | 1（VInt）      |



### 元数据信息

| 字段                 | 描述                                                         | 类型  |
| -------------------- | ------------------------------------------------------------ | ----- |
| numDocs              | fdt文件中保存文档的总数                                      | Int   |
| blockShift           | fdx文件中存储索引数据的block的大小（2的blockShift次方）。范围[2, 22]，默认为10 | Int   |
| totalChuncks + 1     | fdx中的索引数据里存储的数据数量                              | Int   |
| fdxDocsOffset        | 每个chunck中包含多少个文档的信息在fdx文件中存储的起始偏移量  | Long  |
| fdxDocsBlocks        | fdx中保存文档数数据的block的信息                             |       |
| fdxFilePointerOffset | 每个chunck在fdt中的偏移量信息在fdx文件中存储的起始偏移量     | Long  |
| fdxFilePointerBlocks | fdx中保存偏移量数据的block的信息                             |       |
| fdxMaxPointerOffset  | fdx文件中开始存储文件尾的文件偏移量                          | Long  |
| fdtMaxPointerOffset  | fdt文件中开始存储文件尾的文件偏移量                          | Long  |
| numChuncks           | fdt文件中存储的chunck的总数                                  | VLong |
| numDirtyChuncks      | fdt文件中存储的不完整（uncomplete）的chunck的总数            | VLong |

说明：

（1）fdx文件中保存两种信息，第一种是每个chunck包含的文档数目，第二种是每个chunch位于fdt文件中的起始偏移量，这两种数据的包含的数据个数是相同的，并且都为(totalChuncks + 1)，这就是第三个字段的保存的内容。

（2）在没有到达设置的文档数或字节大小而生成Chunck，比如Segment的结束会强行把剩下没处理的文档生成一个Chunck，这种Chunck就是所谓的不完整的Chunck。

（3）数据中有两个保存blocks的字段，这两个字段中的内容是一个数组，数组中的每个元素（对应一个block）的格式如下：

| 字段         | 描述                       | 类型 |
| ------------ | -------------------------- | ---- |
| min          |                            | Long |
| avgIncrement | block中数值的平均增量      | Int  |
| offset       |                            | Long |
| bitsRequired | 保存maxDelta所需的最大位数 | Byte |

字段的具体含义可参考fdx的内容说明。



### 文件尾

| 名称         | 内容                             | 大小(bytes) |
| ------------ | -------------------------------- | ----------- |
| footer_magic | ~CODEC_MAGIC                     | 4           |
| 固定常量     | 0                                | 4           |
| CRC          | 文件中除文件尾之外的内容的校验码 | 8           |