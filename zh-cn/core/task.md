# Task Worker 机制 <!-- {docsify-ignore} -->

受制于 PHP 原生的限制，并非所有 IO 都能够原生的进行异步读写，如文件读写等。
或者 CPU 密集的任务中，若在当前进程中执行，就可能导致其它请求或任务被阻塞。

为了解决这个问题，Wind 框架引入了 Task Worker 机制，通过将同步或 CPU 密集任务发送至一组专用的同步任务进程池中去执行并可获取执行结果，
从而避免了对当前工作进程的阻塞。本质上类似开启了一个新线程进行操作，但实际是在另一个进程中处理。

## 配置

Task Worker 并非必须，需要配置启用 Task Worker。

在 config/server.php 中加入以下配置以启用 Task Worker。

```php
<?php
return [
    /**
     * Channel 服务配置
     * addr 默认为 TCP 通讯方式，绑定地址为 127.0.0.1，可指定为 unix:// 通信方式
     * port 为当使用 TCP 通讯方式时使用的端口
     */
    'channel' => [
        'enable' => true,
        'addr' => 'unix:///tmp/wind-channel.sock',
        //'port' => 2206
    ],
    /**
     * Task Worker 配置
     */
    'task_worker' => [
        /**
         * Task Worker 进程数量
         * 为 0 时将不启动任何 Task Worker 进程
         */
        'worker_num' => 2
    ]
];
```
 
其中 channel 是进程间通讯组件配置，Task Worker 依靠序列化消息进行进程间通信，所以必须启用 Channel 的支持。

task_worker 中需配置 worker_num，配置为多少即为启动多少个 Task Worker 进程。这里根据自己的实际情况配置，
同步任务较多时建议为 CPU 核心数或更多。

配置项 | 必须 | 类型 | 默认值 | 介绍
--- | --- | --- | --- | ---
channel | 是 | 数组 | | Channel 进程间通讯组件配置。
channel.enable | 是 | true | 布尔值 | 是否启用 Channel 进程间通讯组件，建议启用，启用后框架将启动一个 Channel 服务进程。
channel.addr | 否 | 127.0.0.1 | 字符串 | Channel 服务监听地址，支持 TCP 和 Unix Domain Socket 两种形式，指定 IP 地址即为监听 TCP 地址，指定 unix:// 协议的文件名即为使用 Unix Domain Socket 协议，建议使用 Unix Domain Socket 协议。
channel.port | 否 | 2206 | 整型 | 当使用 TCP 形式时所监听的端口。
task_worker | 否 | 数组 | | Task Worker 配置
task_worker.worker_num | 否 | 整型 | 0 | Task Worker 的工作进程数量，数量越多则其能同时执行的同步任务数越多，但相应的内存占用也会更多。

## 使用

使用 Task Worker 来执行同步或 CPU 密集型任务很简单，只需要将对应的闭包和参数通过 TaskWorker 发送即可。

### 使用 Task 类

**\App\MyTask\FooTask 类**
```php
<?php

namespace App\MyTask;

class FooTask {

    public static function take() {
        //Task Worker 执行的闭包同样支持协程环境
        $value = yield asyncOperator();
        //Do something
        return $value;
    }

}
```

**调用 Task Worker 执行**

```php
<?php
$tw = di()->get(\Wind\Task\Task::class);
$result = yield $tw->execute('\App\MyTask\FooTask::take', 'Hello World', 'param 2');
print_r($result);
```

Task 类的 execute 方法第一个参数为要执行的闭包名称，后面跟要传递的参数。

因为 Task Worker 是通过 Channel 使用序列化进程进程通信，所以不能序列化的闭包和值是不能发送至 Task Worker 执行。
如实例化类中的动态方法，或者传递、返回了复杂的对象实例等等。