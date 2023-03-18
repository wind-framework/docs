# 消息队列

## 消息队列的用处

消息队列是一种通过异步解耦来对系统压力进行削峰的机制，是不是没怎么看懂，简单点说就是系统的业务要同时运行压力太大，我们就将它们放到一个排着队的队伍中，让系统再根据自己的能力从里面取出来挨个处理，这样就不至于同时处理导致服务崩溃了。

## 举个栗子

举一个简单的例子，社区派了十个专家在广场上举行答疑解惑的活动，结果来了一千个人围着十个专家提问，七嘴八舌，有些人问了问题专家没听见，有的问题专家听到了但又去解答别人的问题一直没来回答你，然后专家崩溃了，来提问的人看着这乱糟糟的场面也觉得这场活动实在太烂了。

而我们使用消息队列的机制就是让这一千个人先取个号等着，每个专家同一时刻只处理一个人的问题，也就意味十个专家可以同时处理十个人的问题，任何一个专家当前的问题处理完了，就叫下一个号的人上来。因为取号的速度非常快，每个人可能只用了不到一秒就取了一个号，取到号后只需等专家叫到号上前提问即可。

在以上的例子中，十个专家就是消息队列概念中的十个**消费者**，为前来提问的人取号就是**生产者**，一千个人的问题就是要处理的**消息**。

你应该注意到，消息队列并不一定让业务的处理速度变快，但是它能让系统更加有条不紊的处理到来的业务，不至于面对突然到来的大量业务而崩溃，这就是削峰的含义。

## 开始使用

Wind 框架提供了非常简便的机制来让业务在队列中运行，但在此之前需要进行一些配置。

> 消息队列的配置文件为 config/queue.php

### 一、安装

通过 composer 来安装消息队列组件。

```shell
composer require wind-framework/queue
```

### 二、配置队列所使用的驱动

要使用队列，首先要选择一种驱动，队列的驱动是一种存储和传递消息的媒介，它负责消息来到时先排队暂存到哪，以及消息取出等处理。

目前 Wind 框架支持以下几种驱动：
- MySQL - 基于 InnoDB 引擎的数据库驱动。
- Redis - 基于 Redis >= 2.6.0 版本实现。
- Beanstalkd - 一个高性能、轻量级的内存队列系统。

以 Redis 驱动作为示例，首先我们要确保我们的 Redis 连接配置已经OK，在 config/redis.php 中存在以下默认的 Redis 配置：

```php
<?php

// Redis 连接配置
return [
    'default' => [
        // Redis 主机地址
        'host' => '192.168.1.6',
        // Redis 服务端口
        'port' => 6379,
        // Redis 连接密码，没有密码可以为 null
        'auth' => null,
        // 使用的数据库编码
        'db' => 0,
        // 连接超时时间
        'connect_timeout' => 5
    ]
];
```

在确保 Redis 配置无误后，在 config/queue.php 配置文件中配置默认队列：

```php
<?php

return [
    'default' => [
        //指定 RedisDriver 作为驱动
        'driver' => Wind\Queue\Driver\RedisDriver::class,

        // 指定此队列在 Redis 中的键名前缀
        'key' => 'wind:queue-default',

        // 指定 Redis 配置的连接名称，默认为 default
        // 'connection' => 'default'
    ]
];
```

### 三、添加消费者进程

消息队列需要消费者进程来处理消息的任务，所以需要将消息队列的消费者进程添加到自定义进程中来启动消费者。

将默认的消费者进程为 `Wind\Queue\ConsumerProcess::class` 添加到框架的自定义进程配置文件 config/process.php 中。

```php
<?php

return [
    // ...
    Wind\Queue\ConsumerProcess::class,
    // ...
];

```

此时我们的队列就配置完成了。

### 四、创建队列任务

现在消息队列的服务已经配置完成，接下来就是要创建一个任务并且把它放入消息队列中运行了。

比如我们创建一个发送通知的任务 `App\Job\NotificationJob`，任务类需继承自 `Wind\Queue\Job`，该任务包含两个属性，sendTo 代表发送给谁，message 代表要发送的消息内容，通知通过 POST 到 https://www.example.com/api/send 地址来发送。

```php
<?php

namespace App\Job;

use Amp\Http\Client\HttpClient;
use Amp\Http\Client\Body\JsonBody;
use Amp\Http\Client\Request;
use Amp\Http\Client\Response;
use Wind\Queue\Job;

/**
 * 发送通知任务
 */
class NotificationJob extends Job {

    public $sendTo;

    public $message;

    public function __construct($sendTo, $message)
    {
        $this->sendTo = $sendTo;
        $this->message = $message;
    }

    public function handle()
    {
        $sendData = [
            'send_to' => $this->sendTo,
            'message' => $this->message
        ];

        $request = new Request('https://www.example.com/api/send', 'POST');
        $request->setBody(new JsonBody($sendData));

        $httpClient = di()->get(HttpClient::class);

        /** @var $response Response */
        $response = yield $httpClient->send($request);

        if ($response->getStatus() != 200) {
            throw new \Exception('发送失败');
        }
    }

}

```

首先 Job 需要接收相应的任务参数，注意只有 public 和 protected 的属性会被消息队列的驱动存储和传递，private 属性不会被存储，如果使用 private 来存储，那么消息在被消费者进程取到后会丢失 private 的属性。

任务的业务处理代码需写入到 handle() 方法中，handle() 方法已经处于协程环境中，所以不需要使用 call() 函数包裹。

### 五、投递任务

使用 QueueFactory() 获取消息队列的客户端，并调用 put() 方法投递任务，put 方法返回投递后的任务 ID。

```php
use Wind\Queue\Queue;
use Wind\Queue\QueueFactory;

$queueFactory = di()->get(QueueFactory::class);
$queueClient = $queueFactory->get();

$job = new NotificationJob('贾君鹏', '你妈妈喊你回家吃饭啦！');
$jobId = yield $queueClient->put($job);
```

任务投递后，如果消费者进程处于空闲状态就会立即拿到该任务，并执行 handle() 方法来处理该任务。

## 进阶

### 任务的超时限制

如果一个任务运行时间过长，那么任务会堵塞住消费者，导致后续的任务都得不到执行，为了防止这种现象，消息队列默认限制每个任务最长可执行 60 秒，超出这个时间任务将因超进而被踢出消费者，并且进入下一轮重试。

如果你的任务预计运行时间较长，可以在 Job 里定义 ttr 属性的值来指定它的最长运行时间。

```php
class NotificationJob extends Job {

    // 将任务的最长可运行时间调整为 10 分钟
    public $ttr = 600;

}
```

### 任务的失败重试

如果一个任务运行时抛出了一个未被捕获的异常，则消费者会认为该任务失败了，该任务会在之后经过几轮重试，经过多轮重试后仍然失败，则消费者会放弃执行该任务，将该任务置于“失败”状态。

如果想设置任务最多的重试次数，在 Job 中定义 maxAttempts 属性来指定最大重试次数，maxAttempts 默认为2，即任务失败后最多重试两次，如果设置为 0 则代表不重试。

```php
class NotificationJob extends Job {

    // 将任务的最大重试次数调整为 5 次
    public $maxAttempts = 5;

}
```

当一个任务失败时，如果你没有主动捕获该错误，除了会触发状态为 \Wind\Queue\QueueJobEvent::STATE_ERROR 或 \Wind\Queue\QueueJobEvent::STATE_FAILED 的 \Wind\Queue\QueueJobEvent 事件被系统日志记录外，您不会知道它具体失败的原因，如果想要主动捕获任务失败的原因，您可以在 handle() 中套一个大的 try..catch，为了不影响任务的失败判定需要在 catch 中主动抛出任务。

```php
class NotificationJob extends Job {

    public function handle()
    {
        try {
            //..业务代码..
        } catch (\Throwable $e) {
            //得到失败原因 $e->getMessage();
            throw $e;
        }
    }

}
```

如果不想这么写，可以在 Job 中定义 fail() 方法来接收任务失败时的异常，注意如果任务允许重试，则 fail() 仅会在最后进入失败状态时才会触发。

```php
class NotificationJob extends Job {

    public function handle()
    {
        //..业务代码..
    }

	/**
	 * Handle before job into fail queue
	 *
	 * @param Message $message Message object
	 * @param \Throwable $ex The last failed exception
	 * @return bool True into handle queue, or false delete queue
	 */
    public function fail($message, $ex)
    {
        //得到最终失败原因 $ex->getMessage();
    	return true;
    }

}
```

fail() 除了可以接收失败时的异常，还可以通过返回一个布尔值来决定队列之后的状态，true 则进入失败状态，false 则直接删除不再保留。

处于失败状态的任务不会再主动执行，可以通过消费者客户端的 `wakeup(int $num)` 方法来唤醒指定数量的失败任务重新执行，或者使用 `drop(int $num)` 方法来删除指定数量的失败任务。

### 延迟任务

队列客户端 put() 方法支持第二个 delay 参数来指定任务延后多少秒开始执行，在该时间未到之前，队列不会被消费者拿到。

以下是投递一个 60 秒后才开始处理的任务。

```php
$job = new NotificationJob('贾君鹏', '到点了，该回家吃饭了。');
yield $queueClient->put($job, 60);
```

注意指定延迟 60 秒不代表到了该时间后任务一定会执行，这取决于当时消费者是否有空来处理，如果消费者经常不能及时的处理任务，你可以根据所在主机硬件的配置情况适当的增加消费者的数量。

### 取消任务

如果一个任务尚未执行到，或者处于延迟等待的状态时，可以通过消费者客户端的 delete() 方法来删除该任务。

```php
$job = new NotificationJob('贾君鹏', '到点了，该回家吃饭了。');
$jobId = yield $queueClient->put($job, 60);

//贾君鹏已经提前到家，不用再喊了
yield $queueClient->delete($jobId);
```

### 消费者数量

默认情况下，Wind 框架仅会为每个队列启动一个消费者，这显然是不够的，可以根据硬件的资源情况加大消费者的数量。

在队列的配置文件中有两个可选参数 processes 和 concurrent，processes 代表启动多少个消费者进程，concurrent 则指定每个进程里启动多少个消费者，以下是配置默认队列启动 2 个消费者进程，每个进程 16 个消费者的例子，消费者的总数是 processes * concurrent = 32。

```php
<?php

return [
    'default' => [
        //指定 RedisDriver 作为驱动
        'driver' => Wind\Queue\Driver\RedisDriver::class,

        // 指定此队列在 Redis 中的键名前缀
        'key' => 'wind:queue-default',

        // 消费者进程数
        'processes' => 2,

        // 每个消费者进程内的消费者数量
        'concurrent' => 16
    ]
];
```

加大消费者数量会提升处理队列的速度，但是这并不意味着可以随意加大该配置。concurrent 越大，则该进程占用的内存也会越大，另外 processes 加大也是一把双刃剑，更多的进程数代表更高的并行能力，这对磁盘 IO 类是有益的，但是因为是多进程架构，每个进程内的数据库、Http 以及 Redis 等客户端的连接池都是独立的，加大 processes 会导致连接池的复用性下降，整体连接数上涨，所以在增加消费者数量时要进行一定的权衡。


## 队列驱动

### MySQL 驱动
### Redis 驱动
### Beanstalkd 驱动

