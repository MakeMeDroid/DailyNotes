Druid Ingestion随记



IngestionSpec是可以看成是druid入库的规格说明书，其中包含了三类信息：

- DataSchema
- IOConfig
- TuningConfig

其中IOConfig和TuningConfig会根据不同的外部数据源而包含不同的信息。同时，对于任何外部数据源，DataSchema的结构都是相同的。



最终对数据进行写入操作及落盘的接口是`Appenderator`，其唯一的实现类为`AppenderatorImpl`。

处于内存中的segment通过类`IncrementalIndexSegment`表示；当segment落盘后，使用`QueryableIndexSegment`表示。



## Appenderator

Appenderator是向segment中添加数据的接口，其提供了对实时数据的读写功能，所有入库任务的执行流程最终都会汇集到这个接口的add方法：

```java
AppenderatorAddResult add(
    // segment ID
    SegmentIdWithShardSpec identifier,
    // 待写入的一行数据
    InputRow row,
    @Nullable Supplier<Committer> committerSupplier,
    boolean allowIncrementalPersists
) throws IndexSizeExceededException, SegmentNotWritableException;
```



## GenericIndexed

这种类型的数据类似于Map<Integer, Data>，可以通过一个Int类型的Index对数据进行访问。其原理是把Data转化为byte数组依次存储于一块区域中（记为Values），同时通过一个Long数组记录每个Data在Values中的结束偏移量并最终把这个偏移量存储在Header区域中。因为每个偏移量的大小是固定的（8个字节），所有查找第n个Data时，先在Header中获取n-1及n的结束偏移量（如果n为0，则只获取n的结束偏移量），分别为`8 * (n - 1)`和`8 * n`，根据这两个偏移量就可以获取对应Data在Values中的起始偏移量及长度了。

`GenericIndexedWriter`负责数据落盘，`GenericIndexed`负责磁盘文件加载。

存储数据时根据写入数据量的大小使用两个版本的格式。如果数据大小小于一个文件的大小限制则使用版本1，否则使用版本2：

**version 1**

```yaml
format:
  metadata
  header
  values
  
metadata:
  VERSION_ONE(0x1)
  IS_REVERSE_LOOKUP_ALLOWED(0x1 || 0x0)
  totalSize(headerSize + valuesSize + 4)
  numWritten

header:
  endOffset1
  endOffset2
  ...
  endOffsetN

values:
  value1
  value2
  ...
  valueN
  
valueN:
  NULL_VALUE_SIZE_MARKER
  bytes
```

`metadata`：这块数据包含4个信息，首个字节是固定的0x1，表示数据使用的版本1的格式，也就是说只需要读取一个文件；下一个字节表示数据是否是已经排序的，0x1表示已排序，0x0表示未排序，如果数据已排序，可以使用二分法对数据进行查找；然后用一个Int保存数据的字节大小，这个大小包括三部分的内容：header、values和metadata中的numWritten（所以totalSize需要加上一个4字节的大小）；最后的numWritten表示values中保存了多少个数据。之所以numWritten的大小算到totalSize中是为了方便创建`GenericIndexed`时方便。

`header`：header部分记录每个value的结束偏移量（相对values的第一个字节），因为每个偏移量的占用4个字节，所有header的大小为(numWritten * 4)。

`values`：保存每个value的bytes之前，会先存入一个标识，这个标识用一个Int表示，本来是用来存储value的长度的，后来有了header后，存长度就没有意义了，所有用来表示value是不是为null，所有这部分存在空间浪费。标识如果为-1，表示value值为null；如果为0，则表示value为非null。



**version 2**

```yaml
format:
  metadata

metadata:
  VERSION_TWO(0x2)
  IS_REVERSE_LOOKUP_ALLOWED(0x1 || 0x0)
  numValuesPerValueFile
  numWritten
  columnNameLength
  columnNameBytes
```

`numValuesPerValueFile`：使用version2的原因是需要保存的数据的总大小（metadata + header + values）超过了一个文件的大小限制。所有需要把values分隔为多个文件进行存储，numValuesPerValueFile表示每个文件存储多少个value值。这个属性的值是2的幂，并且它的取值能保证每个value的数据不会跨越两个文件。所有这些存储values的文件，除了最后一个文件中存储的value个数可能不足numValuesPerValueFile，其他的都包含numValuesPerValueFile个value。

另外，version2中数据的header和values都使用smoosh进行存储，[smoosh](#Smoosh)的功能和zip类似，可以把多个文件进行压缩，只不过输出多个chunck file。存储到smoosh中的数据都会有一个唯一的名称，header的名称为`<columnName>_header`，而values被分为多个文件，所有它的名称为`<columnName>_value_<N>`。其中`<columnName>`即为metadata中的columnNameBytes记录的内容，例如，对于一个名称为city的维度字段来说，`columnName`的值为`city.dim_values`，所以，header的名称为`city.dim_values_header`，假设values的数据被分为3个文件存储，则这三个文件的名称分别为`city.dim_values_value_0`，`city.dim_values_value_1`，`city.dim_values_value_2`。

smoosh中先存储values的数据，然后存储header的数据。header不会分文件，所以header的大小不能超过一个chunck file的大小限制。header中记录的是每个value在各自文件中的结束偏移量。



## Smoosh

druid中的smoosh是一种保存磁盘数据的文件组织方式。druid需要保存多种类型的数据，每种类型的数据可以看作一个内部文件（internal file），为了减少文件的数量，使用smoosh把这些文件用块文件（chunck file）的形式存储，每个块文件可以包含若干个interval file，一个internal file只会保存到一个chunck file中，也就意味着一个internal file的内容不会跨越多个chunck file。

首先，我们可能需要存储不同类型的数据（比如不同维度的数据），这些数据会被smoosh存储到若干个磁盘文件中，每个文件称为一个块文件（chunck file），每个块文件的大小不超过maxChunckSize（默认为Integer的最大值）。因为不同类型的数据被存储到一起，所有每种类型的数据会有自己的元数据信息，元数据包括四个属性：数据名称（比如维度名），chunckNum，startOffset和endOffset。因为相同类型的数据都是存储一个块文件中的连续空间内，所有每中类型的数据只需要一条元数据信息。

元数据信息存储在名为`meta.smoosh`的文件中，其格式为：

```yaml
file:
  v1,<maxChunckSize>,<chunckFileCount>
  internalFile1_metadata
  internalFile2_metadata
  ...
  internalFileN_metadata

internalFileN_metadata:
  <name>,<chunckFileNum>,<startOffset>,<endOffset>
```

保存数据的块文件的文件名的格式为：`<chunckFileNum>.smoosh`，其中`<chunckFileNum>`是从0开始递增，并且宽度为5的整数，实际的文件名如下：

```yaml
00000.smoosh
00001.smoosh
00002.smoosh
...
```



## ObjectStrategy

ObjectStrategy是一个对数据进行编解码的接口，其内部接口如下：

```java
// 返回进行编解码的数据的java类型
Class<? extends T> getClazz();

// 解码方法：读取字节数据并构建数据对象
T fromByteBuffer(ByteBuffer buffer, int numBytes);

// 编码方法：把数据对象转化为字节数组
byte[] toBytes(@Nullable T val);
```



## Segment Files

不考虑smoosh存储格式的前提下，segment会包含若干文件，有的单独存储，有的存于smoosh中（特别标注）。

Segment的信息文件：

- `version.bin`

  存储IndexIO的版本信息，目前为`9`（int的二级制格式）。

- `factory.json`

  存储`SegmentizerFactory`的实际类型，这个接口用于从磁盘文件加载`Segment`。

- `index.drd` (smoosh)

  包含如下信息：

  - columnNames
  - dimensionNames
  - minTime
  - maxTime
  - bitmapSerdeFactoryType

- `metadata.drd` (smoosh)

  包含`org.apache.druid.segment.Metadata`的JSON表示，其包含如下字段：

  ```java
  Map<String, Object> container;
  AggregatorFactory[] aggregators;
  TimestampSpec timestampSpec;
  Granularity queryGranularity;
  Boolean rollup;
  ```



Segment字段数据(smoosh)：

- <columnName>

  每个字段都会有一个对应的smoosh内部文件，记录字段的数据内容。

  String类型的维度字段存储字典时（使用[GenericIndexed](#GenericIndexed)），如果字典数据太大（超过了一个smoosh文件的最大大小），那么字段数据会分为多个内部文件存储，其中包含一个header文件和若干个values文件：

  - `<dimensionName>.dim_values_header`
  - `<dimensionName>.dim_values_value_0`
  - ...
  - `<dimensionName>.dim_values_value_N`

String类型的维度字段：



## 存储格式

### 压缩方式

压缩方式通过枚举`CompressionStrategy`定义，其包含如下方式：

- **LZF**(0x0)
- **LZ4**(0x1)
- **UNCOMPRESSED**(0xFF)
- **NONE**(0xFE)

注意：dimension类型的字段不能指定为NONE。



### 字段序列化

druid有三种类型的字段：时间、维度和指标。

#### 时间、指标字段

这类指标基本都是Long、Float和Double类型，还有一些特殊的Complex类型。这些类型的数据都会通过接口`GenericColumnSerializer<T>`进行序列化：

```java
public interface GenericColumnSerializer<T> extends Serializer {
    void open() throws IOException;
    
    // ColumnValueSelector可以获取需要存储的值
    void serialize(ColumnValueSelector<? extends T> selector) throws IOException;
}
```

不同的类型有不同的实现类：

1. Long
   - `LongColumnSerializer`
   - `LongColumnSerializerV2`
2. Float
   - `FloatColumnSerializer`
   - `FloatColumnSerializerV2`
3. Double
   - `DoubleColumnSerializer`
   - `DoubleColumnSerializerV2`
4. Complex
   - `ComplexColumnSerializer<T>`
   - `LargeColumnSupportedComplexColumnSerializer<T>`

对于Long、Float和Double这些数值类型，存储时需要考虑要不要存储null值（字段没有值时则为null），当`NullHandling.replaceWithDefault()`为true时，null值会被默认值（Long:0，Double:0.0d，Float:0.0f）代替，也即不存储null，这种情况下会使用不带V2版本的类；否则使用带V2的类序列化数值，其内部会维护一个记录null值所在行数的bitmap来完成对null值的存储。

Complex类型的序列化类通过`ComplexMetricSerde`获取，默认为`ComplexColumnSerializer<T>`，很多自定义的Complex类型使用`LargeColumnSupportedComplexColumnSerializer<T>`进行序列化。两者的区别是，后者可以使用GenericIndexed的多文件功能，但前者只能使用单文件，所有后者存储的数据量可以很大，但前者存储的数据量不能超过一个smoosh文件的大小。



#### 维度字段

维度字段的存储会涉及到索引的合并，所以和指标字段的处理方式不同，处理接口为`DimensionMerger`。

数值类型的维度字段通过`NumericDimensionMergerV9`对数据进行处理，其通过`GenericColumnSerializer`对数据做序列化，并且数据类型与`GenericColumnSerializer`的对应关系和指标字段是一样的，所有数值类型在指标字段中的存储方式与维度字段是相同的。

每种数值类型有自己的Merger类，用于构建序列化数据的`GenericColumnSerializer`：

1. Long

   `LongDimensionMergerV9`

2. Float

   `FloatDimensionMergerV9`

3. Double

   `DoubleDimensionMergerV9`



String类型的维度字段会有对应的倒排索引，其Merger的实现类`StringDimensionMergerV9`需要处理字典数据，bitmap索引数据，以及编码后的字段数据等。



### Long

序列化接口：`ColumnarLongsSerializer`

实现类：

`IntermediateColumnarLongsSerializer`

`EntireLayoutColumnarLongsSerializer`

`BlockLayoutColumnarLongsSerializer`



反序列化接口：`ColumnarLongs`

反序列化接口通过`Supplier<ColumnarLongs>`提供，获取到磁盘数据后，先创建`CompressedColumnarLongsSupplier`，其会根据解析出的元数据信息决定创建`EntireLayoutColumnarLongsSupplier`或`BlockLayoutColumnarLongsSupplier`。



Long型数据的编码主要有三种方式：

- **DELTA**(0x0)

  原理是只存储数值中的最小值以及每个数值与最小值之间的差值。实际过程中，先获取待存储数值中的最大最小值，然后把最大值与最小值之间的差值记为maxDelta，存储的每个差值的位数不会超过存储maxDelta所需的位数，因此对差值的存储可以采用`VSizeLongSerde`定义的组织方式（在Lucene中称为PackedInt）。

- **TABLE**(0x1)

  如果待存储数值的基数为N，并且N小于特定值（Druid中为256，记为maxTableSize），可以先对这些数值进行编码使其对应到数字[0, N - 1]（原理与String类型的字典一样）。这样只需要存储N个不同的数字以及每个数值对应的编码值，当需要存储的数值个数远远大于N时，这种方式可以有效的节省存储空间，因为每个数值只需要1个字节的存储空间（在maxTableSize为256的情况下）。

- **LONGS**(0xFF)

  每个数值用8个字节依次存储，也就是没用任何压缩技巧。



上述的编码方式存储时都会记录元数据信息以便解码时能确定使用哪种方式，元数据信息的内容如下：

**DELTA**

```yaml
metadata:
  compression_strategy_encoding_flag
  delta_encoding_format_id
  delta_encoding_version
  minValue
  bitsPerValue

compression_strategy_encoding_flag：
  compression_strategy_id - FLAG_VALUE

FLAG_VALUE：
  126
```

`compression_strategy_encoding_flag`：对CompressionStrategy的id做编码后的结果。做这一步操作是为了向后兼容，最开始Druid只支持LONGS编码方式，其元数据中只存储了CompressionStrategy的id信息，没有用来区分不同编码方式的信息，所有这里把CompressionStrategy的id转化为小于-2(0xFE，NONE的id)的值。当读取数据时，如果小于-2，则表示其后跟着编码的具体信息；否则说明编码方式为LONGS。至于这里为啥FLAG_VALUE为126，是因为当前CompressionStrategy的最小id为-2(0xFE)，要保证转化后的值能用一个byte表示，也即转化后的最小值为-128，所有FLAG_VALUE为(-2 - (-128))，当然这里FLAG_VALUE取[4, 126]中的值任意一个值也是可以的，只不过用126可以表示尽可能多的值。

`delta_encoding_format_id`：编码枚举中DELTA的ID，值为0。

`delta_encoding_version`：delta编码的版本号，当前为1。

`minValue`：所有数值中的最小值。

`bitsPerValue`：存储每个delta需要的位数。



**TABLE**

```yaml
metadata:
  compression_strategy_encoding_flag
  table_encoding_format_id
  table_encoding_version
  table_size
  values_sorted_by_index
```

`compression_strategy_encoding_flag`：同DELTA。

`table_encoding_format_id`：编码枚举中TABLE的ID，值为0x1。

`table_encoding_version`：table编码的版本号，当前为0x1。

`table_size`：table相当于一个Map<number, code>，其number为待存储的数值，而code是[0, N - 1]之间的值，table_size就是这个Map的大小。

`values_sorted_by_index`：把table看作一个Map<number, code>，这部分内容是按code排序的number值列表。



**LONGS**

```yaml
metadata:
  compression_strategy_id
```

LONGS是最初默认的方式，只简单存储压缩方式的ID值。



怎么选择上面的编码方式目前有两种策略：

- **AUTO**

  先获取所有的数值，然后根据数值表现出的特性决定采用哪种编码方式。优先考虑TABLE方式，如果不满足条件则考虑DELTA方式，如果还不满足则使用LONGS方式。

- **LONGS**

  始终使用LONGS编码方式。



确定了编码方式后，还需要根据压缩方式来决定最终的Serializer：

如果压缩方式为NONE，则使用`EntireLayoutColumnarLongsSerializer`，否则使用`BlockLayoutColumnarLongsSerializer`。

- `EntireLayoutColumnarLongsSerializer`

  简单的逐个数字进行存储。

  元数据信息：

  ```yaml
  metadata:
    version
    value_total_count
    0
    long_encoding_metadata
  ```

  `version`：当前为0x2

  `value_total_count`：存储的数值总个数

  `0`：固定常量，与`BlockLayoutColumnarLongsSerializer`的metadata中的`value_count_per_block`对应。

  `long_encoding_metadata`：Long编码方式的metadata，可以是DELTA、TABLE和LONGS中的一种。

  

- <a name="BlockLayoutColumnarLongsSerializer">`BlockLayoutColumnarLongsSerializer`</a>

  按block对数值进行存储的组织方式，每个block的大小不超过64K。block是执行压缩算法的数据单元，压缩后的数据通过[GenericIndexed](#GenericIndexed)存储，配合元数据中的`value_count_per_block`可以获取任意的第n个数值。

  元数据信息：

  ```yaml
  metadata:
    version
    value_total_count
    value_count_per_block
    long_encoding_metadata
  ```

  `version`：当前为0x2

  `value_total_count`：存储的数值总个数。

  `value_count_per_block`：每个block中可以存储的数值个数。

  `long_encoding_metadata`：同上。



### Float

序列化接口：`ColumnarFloatsSerializer`

有两种编码实现：

- <a name="EntireLayoutColumnarFloatsSerializer">`EntireLayoutColumnarFloatsSerializer`</a>

  当compression为NONE时使用这种格式，内部依次连续得存储每个float转化为int后的值（`Float.floatToRawIntBits`）。

  元数据信息：

  ```yaml
  metadata：
    version
    value_total_count
    0
    compression_NONE_id
  ```

  `version`：0x2

  `value_total_count`：存储的数值总个数。

  `0`：固定常量，与`BlockLayoutColumnarFloatsSerializer`的metadata中的`value_count_per_block`对应。

  `compression_NONE_id`：也是固定值，`CompressionStrategy.NONE`的id值。

  

- <a name="BlockLayoutColumnarFloatsSerializer">`BlockLayoutColumnarFloatsSerializer`</a>

  与[BlockLayoutColumnarLongsSerializer](#BlockLayoutColumnarLongsSerializer)的原理相同，只不过存储的是float类型的数值。
  
  元数据信息：
  
  ```yaml
  metadata:
    version
    value_total_count
    value_count_per_block
    compression_id
  ```
  
  `version`：0x2
  
  `value_total_count`：存储的数值总个数。
  
  `value_count_per_block`：每个block中可以存储的数值个数，值为`(blockSize / Float.BYTES)`。
  
  `compression_id`：`CompressionStrategy`的id值。



反序列化接口：`ColumnarFloats`

反序列化接口通过`Supplier<ColumnarFloats>`提供，获取到磁盘数据后，先创建`CompressedColumnarFloatsSupplier`，其会根据解析出的元数据信息决定创建`EntireLayoutColumnarFloatsSupplier`或`BlockLayoutColumnarFloatsSupplier`。



### Double

序列化接口：`ColumnarDoublesSerializer`

有两种编码方式：

- `EntireLayoutColumnarDoublesSerializer`

  compression为NONE时使用此格式，依次连续的保存每个double值。

  元数据信息：

  ```yaml
  metadata：
    version
    value_total_count
    0
    compression_NONE_id
  ```

  内容同[EntireLayoutColumnarFloatsSerializer](#EntireLayoutColumnarFloatsSerializer)

  

- `BlockLayoutColumnarDoublesSerializer`

  与[BlockLayoutColumnarLongsSerializer](#BlockLayoutColumnarLongsSerializer)的原理相同，只不过存储的是double类型的数值。

  元数据信息：

  ```yaml
  metadata:
    version
    value_total_count
    value_count_per_block
    compression_id
  ```

  内容同[BlockLayoutColumnarFloatsSerializer](#BlockLayoutColumnarFloatsSerializer)。其中`value_count_per_block`的值为`(blockSize / Double.BYTES)`。



反序列化接口：`ColumnarDoubles`

反序列化接口通过`Supplier<ColumnarDoubles>`提供，获取到磁盘数据后，通过`CompressedColumnarDoublesSuppliers`类的静态方法解析出的元数据信息并决定创建`EntireLayoutColumnarDoublesSupplier`或`BlockLayoutColumnarDoublesSupplier`。



### String

维度字段通常为String类型，这种类型的字段会存储索引，并且允许多值输入，所以其需要处理的内容会比其它类型多，包括如下内容：

1. 字典信息
2. 字段值（进过字典编码）
3. bitmap信息
4. spatial数据(可选)



#### 字典

向`IncrementalIndex<AggregatorType>`类中添加行数据时，行中的维度字段会先被处理，对于String类型的字段来说就是构建字典（对于数值类型的维度字段并不构建字典）。

字典构建由`DimensionIndexer`中的`processRowValsToUnsortedEncodedKeyComponent`完成：

```java
public interface DimensionIndexer<EncodedType extends Comparable<EncodedType>,
                                  EncodedKeyComponentType,
                                  ActualType extends Comparable<ActualType>> {
  // dimValues是维度字段的原始值，这个方法对原始值进行编码，并返回编码后的结果。
  // String类型的字段使用字典进行编码；数字维度字段则不进行编码，返回原始值
  EncodedKeyComponentType processRowValsToUnsortedEncodedKeyComponent(@Nullable Object dimValues,
                                                                      boolean reportParseExceptions);
  
  // 计算编码后的值占用的内存空间，包括字典中的空间大小
  long estimateEncodedKeyComponentSize(EncodedKeyComponentType key);
  
  // 把编码后的值还原为维度的原始值
  Object convertUnsortedEncodedKeyComponentToActualList(EncodedKeyComponentType key);
  
  /** 维度字段接下来的操作都是基于编码之后的值进行的，下面的函数提供了比较及计算hash值的功能 **/
  
  // 比较两个编码后的字段值
  int compareUnsortedEncodedKeyComponents(@Nullable EncodedKeyComponentType lhs,
                                          @Nullable EncodedKeyComponentType rhs);
  
  // 判断两个编码后的字段值是否相等
  boolean checkUnsortedEncodedKeyComponentsEqual(@Nullable EncodedKeyComponentType lhs,
                                                 @Nullable EncodedKeyComponentType rhs);
  
  // 通过编码后的字段值计算hashcode
  int getUnsortedEncodedKeyComponentHashCode(@Nullable EncodedKeyComponentType key);
}
```

`DimensionIndexer`中的类型参数：

`EncodedType`：对维度字段编码后的值的类型，通常为`Integer`。

`EncodedKeyComponentType`：字段数据通过字典编码后的最终值，因为String类型的维度字段运行多值，所以类型为`int[]`。

`ActualType`：字段实际的类型，String维度字段就是`String`。

`processRowValsToUnsortedEncodedKeyComponent`函数本身也没有规定必须用字典对维度数据进行编码，所有可以用任何可用的方式进行处理。String类型的维度字段会对字段值去重并为每个去重后的值指定一个int类型的编号，这个过程通过一个Map即可完成，这个Map就是所谓的字典，它其实是一种压缩手段，减少String类型维度数据的存储空间。

内存中用Map存储字典是一个很自然并且可行的方法，因为Map可以很高效的实现通过维度值获取其对应编号的功能，但字典最终是要随着Segment存储到磁盘的，用磁盘存储Map是一个复杂的过程（KV结构数据存储），所有存储到磁盘前，Druid对字典做了下转换。原理是对字典中的Key（即维度值）进行排序，然后把Key对应的编号变更为Key位于排序列表中的位置，例如，如果原始字典为：

`{"b" -> 0, "c" -> 1, "a" -> 2}`

则处理后的字典为：

`{"a" -> 0, "b" -> 1, "c" -> 2}`

这样处理后有两个好处，一方面，存储时只需要存储Key值，节省了存储编号的空间，还可以通过对Key值进行二分查找来高效的实现Map的功能。另一方面，Segment之间会进行合并，因此字典也会合并，已经排序的Key值可以很方便的完成字典的合并。

需要注意的是，排序处理只能在没有新数据写入的情况下进行，这时候所有的维度值已经明确，但前面叙述的对维度值进行编码的过程是在数据写入过程中完成的，所以维度的编码值是未排序字典中的编码，但最终我们需要存储的又是排序之后的字典的编码，所有`DimensionIndexer`会提供一些方法实现未排序编码与已排序编码之间的转换，如下：

```java
// 获取排序字典中的编码值对应到未排序字典中的编码值
// 用上面的例子来说，如果此处传入0，也就是"a"在排序字典中的编码值，则函数会返回2，也即"a"在未排序字典中的编码值
EncodedType getUnsortedEncodedValueFromSorted(EncodedType sortedIntermediateValue);

// 获取已排序字典中的维度值
CloseableIndexed<ActualType> getSortedIndexedValues();

// 把未排序字典编码的维度值转化为已排序字典的编码的维度值
ColumnValueSelector convertUnsortedValuesToSorted(ColumnValueSelector selectorWithUnsortedValues);
```



当不同Segment进行合并时，字典也需要合并，这时候纬度值对应的编码又会重新构建，这个构建过程由`DimensionMerger`类的如下方法完成：

```java
// 合并多个Segment中的维度字段的字典。参数中每个IndexableAdapter代表一个Segment。
void writeMergedValueDictionary(List<IndexableAdapter> adapters) throws IOException;
```

每个维度字段会有一个`DimensionMerger`对象，用于合并此维度字段的数据。因为每个Segment中的字典已经按Key进行排序了，所以字典合并是通过优先队列对各个字典的Key进行合并，然后记录下每个segment的字典中的每个Key的原始编码值到合并后编码值之间的映射关系。

合并后的字典通过[GenericIndexed](#GenericIndexed)进行存储。



#### 字段值

String类型的维度字典可以有多值，当字段有多个值时，这些值会当做数组处理。字段值存储前会先通过字典进行编码，然后存储其编码后的结果，比如，如果某个字段的两行数据为：

```
"cat"
["dog", "cat"]
```

其中，第二行是多值，所有作为数组对待。存储前先对其建立字典（已排序之后的），如下：

```
"cat" -> 0
"dog" -> 1
```

则字段值通过字典编码后变为：

```
0
[1, 0]
```

这就是最终存储到磁盘上的维度值。接下来看看这些值怎么存储到磁盘上。

怎么存储字段值需要考虑两个因素：维度字段的压缩策略以及字段中是否含有多值。

1. 包含多值并且不压缩数据

   `VSizeColumnarMultiIntsSerializer`

2. 包含多值并且压缩数据

   `V3CompressedVSizeColumnarMultiIntsSerializer`

3. 不包含多值并且不压缩数据

   `VSizeColumnarIntsSerializer`

4. 不包含多值并且压缩数据

   `CompressedVSizeColumnarIntsSerializer`



字段值的序列化接口为`ColumnarIntsSerializer`，其实现类如下

```
ColumnarIntsSerializer
  |_ ColumnarMultiIntsSerializer
  |  |_ VSizeColumnarMultiIntsSerializer
  |  |_ V3CompressedVSizeColumnarMultiIntsSerializer
  |
  |_ SingleValueColumnarIntsSerializer
     |_ VSizeColumnarIntsSerializer
     |_ CompressedColumnarIntsSerializer
     |_ CompressedVSizeColumnarIntsSerializer
```

- <a name="VSizeColumnarIntsSerializer">`VSizeColumnarIntsSerializer`</a>

  原理是先获取表示数据中最大值需要的最少字节数numBytes，然后每个数字均用numBytes个字节存储。

  元数据信息：

  ```yaml
  metadata:
    VSizeColumnInts_version
    numBytes
    total_byte_count
  ```

  `VSizeColumnInts_version`：0x0

  `numBytes`：用于存储每个数字的字节数，值位于[1, 4]，所有用一个byte存储。

  `total_byte_count`：存储所有数字占用的字节数，值为`(numBytes * valuesCount)`。

  

- <a name="VSizeColumnarMultiIntsSerializer">`VSizeColumnarMultiIntsSerializer`</a>

  因为字段数据中有数组（多值），所有需要保存每个字段数据的结束偏移量，偏移量信息位于header区域中，字段数据存储于values区域中。header中把每个偏移量存储为4字节的整数，values中的数据存储原理和[VSizeColumnarIntsSerializer](#VSizeColumnarIntsSerializer)一样。

  元数据信息：

  ```yaml
  metadata:
    VSizeColumnarMultiInts_version
    numBytes
    total_size
    dimension_value_count
  ```

  `VSizeColumnarMultiInts_version`：0x1

  `numBytes`：values区域中存储每个值使用的字节个数。

  `total_size`：值为`(headerSize + valuesSize + Integer.BYTES)`。

  `dimension_value_count`：values中包含的维度值的个数。

  

- <a name="CompressedVSizeColumnarIntsSerializer">`CompressedVSizeColumnarIntsSerializer`</a>

  先把数据分为block，每个block包含chunkFactor个维度值，并且每个block的大小不超过64K。block中数据的存储方式和`VSizeColumnarIntsSerializer`相同。然后通过指定的压缩算法对每个block进行压缩存储，压缩后的数据使用[GenericIndexed](#GenericIndexed)存储。

  元数据信息：

  ```yaml
  metadata：
    CompressedVSizeColumnarInts_version
    numBytes
    dimension_value_count
    chunkFactor
    compression_id
  ```

  `CompressedVSizeColumnarInts_version`：0x2

  `numBytes`：block中存储每个数值使用的字节个数。

  `dimension_value_count`：总共存储了多少个维度值。

  `chunkFactor`：每个block中保存的数值个数。

  `compression_id`：压缩算法的[id](#压缩方式)。

  

- <a name="V3CompressedVSizeColumnarMultiIntsSerializer">`V3CompressedVSizeColumnarMultiIntsSerializer`</a>

  和[VSizeColumnarMultiIntsSerializer](#VSizeColumnarMultiIntsSerializer)的情况一样，需要保存header和values两种数据，不同的是header中保存的是每个维度值中第一个数值的位置偏移量，举个栗子，假如需要存储三行数据：

  ```
  [1, 2]
  [2]
  [0, 1, 3]
  ```

  那么，把所有数据看作一个数组`[1, 2, 2, 0, 1, 3]`，位置偏移量就是每个维度的第一个值在这个数组中的索引值，第一行的是0，第二行是2，第三行是3。

  另外，header中的数据使用[CompressedColumnarIntsSerializer](#CompressedColumnarIntsSerializer)存储，而values中的数据使用[CompressedVSizeColumnarIntsSerializer](#CompressedVSizeColumnarIntsSerializer)存储。

  元数据信息：

  ```yaml
  metadata:
    V3CompressedVSizeColumnarMultiInts_version
  ```

  `V3CompressedVSizeColumnarMultiInts_version`：0x3

  

- <a name="CompressedColumnarIntsSerializer">`CompressedColumnarIntsSerializer`</a>

  把数值分为block，然后对block进行压缩并用[GenericIndexed](#GenericIndexed)存储。block中每个数值占用4字节。这个类用于存储[V3CompressedVSizeColumnarMultiIntsSerializer](V3CompressedVSizeColumnarMultiIntsSerializer)中的header信息。

  元数据信息：

  ```yaml
  metadata:
    CompressedColumnarInts_version
    value_count
    chunkFactor
    compression_id
  ```

  `CompressedColumnarInts_version`：0x2

  `value_count`：总共存储了多少数值

  `chunkFactor`：每个block中存储多少数值

  `compression_id`：压缩算法的[id](#压缩方式)。



字段值的反序列化接口分为两个：

- `ColumnarInts`
- `ColumnarMultiInts`



#### Bitmap

维度字段的字典中的每个Key都会对应一个bitmap数据，bitmap中保存了Segment中的对应维度字段包含Key值的所有行号。bitmap数据通过[GenericIndexed](#GenericIndexed)存储，因为字典中的Key和bitmap一一对应，所以如果想获取某个Key的bitmap，现在字典中查找Key，取得Key在字典中的编号后，根据这个编号就可以获取bitmap了。