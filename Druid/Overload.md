## Task Queue

任务的提交接口与任务的执行者之间有一个中间地带，也即`TaskQueue`。

TaskQueue主要做三个工作：

1. 接收提交者提交的任务，并将其放入任务列表，并向`TaskStorage`保存任务
2. 定时检测任务列表中新增的任务，并通过`TaskRuner`对它们进行执行，并清理已经从任务列表中移除，但还在执行的任务
3. 定时把`TaskStorage`中记录的任务同步到任务列表。TaskStorage中的内容可能被外部修改，而集群中需要执行的任务应该以TaskStorage的内容为主，TaskQueue需要定期添加TaskStorage中新增的任务到任务列表，并从任务列表中删除TaskStorage中没有的任务。



## Task Storage

外界对Task的操作可以有两种途径：

- HTTP接口
- Metadata数据库表

无论哪种方式，最终的目的都是修改`TaskQueue`中的`tasks`列表。

Task被提交到TaskQueue之前需要先以RUNNING状态写入TaskStorage。当Task的状态发送变更时，状态信息会被存储到TaskStorage上。



## RemoteTaskRunner

默认实现通过ZK进行任务的下发及状态获取。基于HTTP的`HttpRemoteTaskRunner`抛弃了ZK，使用HTTP连接的方式进行任务下发以及状态实时获取。

HttpRemoteTaskRunner监听任务状态的方式是通过一个阻塞的Http请求，当有任务信息变更时，请求才会返回。