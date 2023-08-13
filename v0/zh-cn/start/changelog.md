# 更新日志

## 日志 `0.2.0`

更新日期: 2023/6/6

1. 彻底重构了异步日志机制

    之前 Logger 日志由异步 Handler 转交给 TaskWorker 进程，然后 Task Worker 调用 Logger 使用同步 Handler 处理一遍，此机制存在 Logger 在原进程和 TaskWorker 中双重调用，从而导致日志分组的 Handler、Processor 也双重处理、日志被 Logger 两次处理导致关键信息（如时间）不准确等问题。且为了区分所以进程与环境，也导致了实现的复杂性。

    现在使用了全新的机制构建异步和同步 Handler，并且转交给 TaskWorker 后不再由 Logger 进行处理，而是直接交给同步 Handler，避免了各种双重处理，保证了日志与发起时相同，性能也有所提升，已经完全生产可用。

2. 新增异步日志专用写入进程选项（LogWriterProcess）

    现在新增了另一选项：使用专用独立进程写入异步日志，专用进程能够更进一步减少和评估日志 IO 对系统带来的影响。

3. 异步日志继续保持了 `async` 选项对任意 Handler 异步化的能力。

## 数据库 `0.2.0`

更新日志: 2023/3/20

1. 支持简单 ORM 机制（Model）

    现在提供了简易的数据模型（Model）功能，一个模型映射一个数据表，支持主键 find()，支持基于查询构建造的检索与更新、完备的字段自增自减、虚拟字段、数组式访问。支持原始属性、脏属性、已更新属性的检索等。

    未来将进一步支持模型事件、钩子机制，为方便易用的数据库操作打下基础。

## 消息队列 `0.2.0`

更新日期: 2023/3/17

1. 新增队列任务在运行中重置任务超时时间的功能（增加任务的执行时间）。

    原先任务的可执行时间上限由 `ttr` 属性控制，任务一旦创建并开始执行，限制便开始计时，如果到达时间任务仍未完成，会被消费者进程以超时失败处理，任务之后等待重试或进入失败状态。如果任务知道自己仍然在正常运行，但是需要更多的时间才能处理完成，现在可以调用 `touch()` 方法来让消费者进程重新计时，从而延长当前任务的执行时间，这对于需要长时间执行的任务很有帮助。

2. Redis 驱动使用了更多的事务，以进一步保证异常情况下的数据原子性。
3. Beanstalk 驱动的一些错误修复。

## Web `0.3.2`

更新日期: 2022/8/3

1. WebSocket 服务器新增配置支持心跳 ping/pong 的回调。
2. 新增选项用于开启 Web 服务的 SSL 功能。
3. 新增选项以支持对不同 Web 服务指定不同的路由配置。

更早的更新日志，之前没写...