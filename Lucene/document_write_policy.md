Lucene的flush机制可以通过接口`org.apache.lucene.index.FlushPolicy`进行定义，目前Lucene中唯一的实现是`FlushByRamOrCountsPolicy`，其可以通过当前已经写入的doc数以及当前使用的内存数为依据进行Flush操作。

- document数量

  检测对象为`DocumentsWriterPerThread`，如果DocumentsWriterPerThread对象中已经写入的document总数超过了config中的`maxBufferedDocs`（默认为-1，也就是禁用），此DocumentsWriterPerThread会被标记为FlushPending。

- 内存使用量

  检测对象为`DocumentsWriterFlushControl`，如果DocumentsWriterFlushControl中记录的内存使用量（activeBytes + deleteBytes）超过了config中的`ramBufferSizeMB`（默认为16），则选择DocumentsWriterFlushControl中内存使用量最大的DocumentsWriterPerThread对象，并对其标记为FlushPending。



除了上述自定义的Flush规则外，如果DocumentsWriterPerThread对象的内存使用量超过了config的`perThreadHardLimitMB`（默认为1945），则此DocumentsWriterPerThread对象也会被标记为FlushPending。