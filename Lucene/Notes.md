每个Token都会经过Indexing Chain(DefaultIndexingChain)的处理，这个链其实是TermsHash的链，另外还包含一个StoredFieldsConsumer，根据Segment是否需要排序分为两种链：

- 有序链

  **FreqProxTermsWriter** -> **SortingTermVectorsConsumer**

  StoredFieldsConsumer[**SortingTermVectorsConsumer**]

- 无序链

  **FreqProxTermsWriter** -> **TermVectorsConsumer**

  StoredFieldsConsumer[**TermVectorsConsumer**]



# Codec

Lucene中的Codec接口定义了存储数据的格式，可以通过JDK的ServiceLoader进行注册，然后通过名称被引用。

可以通过实现Codec中的接口返回如下数据格式：

- PostingsFormat
- DocValuesFormat
- StoredFieldsFormat
- TermVectorsFormat
- FieldInfosFormat
- SegmentInfoFormat
- NormsFormat
- LiveDocsFormat
- CompoundFormat
- PointsFormat

所有这些Format都是对某类数据进行序列化与反序列化，以便通过磁盘文件进行读写。
