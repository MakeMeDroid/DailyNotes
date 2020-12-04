## Resource

- TaskManagementResource

  负责接收Task，并通过`WorkerTaskManager`进行管理，并且提供获取任务状态变化的接口。

  WorkerTaskManager把任务分为三种：assigned、running和completed

- WorkerResource

  提供修改MiddleManager节点状态、获取任务ID列表、关闭任务及获取任务日志的接口。

- ShuffleResource



WorkerCuratorCoordinator

zk路径：

baseAnnouncementsPath：`/druid/indexer/announcements/<worker_host>`

baseTaskPath：`/druid/indexer/tasks/<worker_host>`

baseStatusPath：`/druid/indexer/status/<worker_host>`



baseAnnouncementsPath再zk上保存`Worker`的信息：

```yaml
Worker:
  scheme         #String
  host           #String
  ip             #String
  capacity       #int
  version        #String
  category       #String
```

这里的Worker指MiddleManager节点。



statusPath：`<baseStatusPath>/<statusId>`

存储`TaskAnnouncement`：

```yaml
TaskAnnouncement：
  id                      #taskId
  type                    #taskType
  status                  #TaskState
  taskResource            #TaskResource
  taskLocation            #taskLocation
  taskDataSource          #taskDataSource
```

