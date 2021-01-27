# Task Meta

## druid_tasks

| 字段名         | JAVA类型   | 说明                 |
| -------------- | ---------- | -------------------- |
| id             | String     | 任务ID               |
| created_date   | DateTime   | 任务创建时间         |
| datasource     | String     | 任务所属的DataSource |
| payload        | Task       | 任务信息             |
| active         | Boolean    | 任务是否处于运行状态 |
| status_payload | TaskStatus | 任务的状态信息       |

Overload节点中，任务被添加到TaskQueue之后，任务的信息便被添加到这个表中。



## druid_tasklogs

| 字段名      | JAVA类型      | 说明                     |
| ----------- | ------------- | ------------------------ |
| id          | Long          | 关系数据库中的自增ID     |
| task_id     | String        | 任务ID                   |
| log_payload | TaskAction<T> | 任务运行过程中执行的操作 |

每个任务运行过程中可能会执行若干个TaskAction，这个表用于记录任务执行过的Action历史。



## druid_tasklocks

| 字段名       | JAVA类型 | 说明                   |
| ------------ | -------- | ---------------------- |
| id           | Long     | 关系数据库中的自增ID   |
| task_id      | String   | 任务ID                 |
| lock_payload | TaskLock | 任务运行过程中获取的锁 |

记录每个任务执行过程中获取到的锁。



# Data Source Metadata

## druid_datasource

| 字段名                  | JAVA类型           | 说明                                                      |
| ----------------------- | ------------------ | --------------------------------------------------------- |
| dataSource              | String             | dataSource名                                              |
| created_date            | DateTime           | 记录创建时间                                              |
| commit_metadata_payload | DataSourceMetadata | DataSource的元数据（对于Kafka任务，保存每个分区的偏移量） |
| commit_metadata_sha1    | String             | 元数据的sha1编码（Base16表示）                            |



## druid_segments

| 字段名       | JAVA类型    | 说明                         |
| ------------ | ----------- | ---------------------------- |
| id           | String      | Segment的ID                  |
| dataSource   | String      | Segment所属的dataSource      |
| created_date | DateTime    | 记录创建时间                 |
| start        | DateTime    | Segment所在区间的开始时间    |
| end          | DateTime    | Segment所在区间的结束时间    |
| partitioned  | Boolean     | Segment是否有分片(ShardSpec) |
| version      | String      | Segment的version             |
| used         | Boolean     | (**不详**)                   |
| payload      | DataSegment | Segment的详细信息            |



## druid_pendingsegments

| 字段名                     | JAVA类型               | 说明                          |
| -------------------------- | ---------------------- | ----------------------------- |
| id                         | String                 | Segment的ID                   |
| dataSource                 | String                 | Segment所属的DataSource       |
| created_date               | DateTime               | 记录创建时间                  |
| start                      | DateTime               | Segment的时间区间的开始时刻   |
| end                        | DateTime               | Segment的时间区间的结束时刻   |
| sequence_name              | String                 | Segment所属任务的sequenceName |
| sequence_prev_id           | String                 |                               |
| sequence_name_prev_id_sha1 | String                 |                               |
| payload                    | SegmentIdWithShardSpec |                               |

