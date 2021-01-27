## Task Paths

`TaskConfig`可以配置与Task相关的路径，其相关属性在Druid的配置文件中设置：

- baseDir

  配置项：`druid.indexer.task.baseDir`

  默认值：JVM属性`java.io.tmpdir`的值。

- baseTaskDir

  配置项：`druid.indexer.task.baseTaskDir`

  默认值：`<baseDir>/persistent/task`

- hadoopWorkingPath

  配置项：`druid.indexer.task.hadoopWorkingPath`

  默认值：`/tmp/druid-indexing`



上面三个是用户可配置的路径，与具体任务有关的路径：

- TaskDir

  路径：`<baseTaskDir>/<taskId>`

- TaskWorkDir

  路径：`<baseTaskDir>/<taskId>/work`

- TaskTempDir

  路径：`<baseTaskDir>/<taskId>/temp`

- TaskLockFile

  路径：`<baseTaskDir>/<taskId>/lock`



`TaskToolBox`中定义的任务路径：

- TaskIndexingTmpDir

  路径：`<baseTaskDir>/<taskId>/work/indexing-tmp`

- TaskMergeDir

  路径：`<baseTaskDir>/<taskId>/work/merge`

- TaskPersistDir

  路径：`<baseTaskDir>/<taskId>/work/persist`



## AppenderatorImpl

`BasePersistDirectory`：`<baseTaskDir>/<taskId>/work/persist`

CommitFile：`<baseTaskDir>/<taskId>/work/persist/commit.json`

LockFile：`<baseTaskDir>/<taskId>/work/persist/.lock`

sequencesFile：`<baseTaskDir>/<taskId>/work/persist/sequences.json`

Segment相关目录及文件：

SegmentPersistDir：`<baseTaskDir>/<taskId>/work/persist/<segment_identifier>`

SegmentIdentifierFile：`<baseTaskDir>/<taskId>/work/persist/<segment_identifier>/identifier.json`

SegmentDescriptorFile：`<baseTaskDir>/<taskId>/work/persist/<segment_identifier>/descriptor.json`



每个Segment对应一个`Sink`，每个sink有一个SegmentPersistDir，sink中会包含若干个`FireHydrant`，每个FireHydrant维护这个Segment中的部分数据，并且同一时刻只有一个FireHydrant处于可写状态，当可写的FireHydrant被写入磁盘后，它便会作为一个历史FireHydrant在Sink中维护，同时一个新的可写的FireHydrant会被创建并用于写入。每个FireHydrant被分配一个从0递增的ID，在SegmentPersistDir目录下会有一个名称与ID相同的目录用于存储FireHydrant维护的数据。

FireHydrant在持久化时，CommitFile用于记录每个sink中已经执行持久化的FireHydrant的数量，当任务重启时，可以根据这些信息恢复已经写成功的数据。除了记录FireHydrant的信息外，CommitFile还会记录与任务类型相关的元数据信息。