# 消息队列

## 消息队列的用处

消息队列是一种通过异步解耦来对系统压力进行削峰的机制，是不是没怎么看懂，简单点说就是系统有大量的业务要同时处理时压力太大，我们就将它们放到一个排着队的队伍中，让系统再根据自己的能力从里面挨个取出来处理，这样就不至于同时处理导致服务崩溃了。

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

要使用队列，首先要选择一种驱动，队列的驱动是一种存储和传递消息的媒介，它负责消息来到时先排队暂存到哪，以及消息取出和各种原子性保证等。

目前 Wind 框架支持以下几种驱动：
- MySQL - 基于 InnoDB 引擎的数据库驱动，适合在较低负载下使用。
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

将默认的消费者进程 `Wind\Queue\ConsumerProcess::class` 添加到框架的自定义进程配置文件 config/process.php 中。

```php
<?php

return [
    // ...
    Wind\Queue\ConsumerProcess::class,
];

```

到这我们的队列就配置完成了。

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
        $response = $httpClient->request($request);

        if ($response->getStatus() != 200) {
            throw new \Exception('发送失败');
        }
    }

}

```

首先 Job 需要接收相应的任务参数，注意只有 public 和 protected 的属性会被消息队列的驱动存储和传递，private 属性不会被存储，消息在放入队列后就会丢失 private 的属性。

任务的业务处理代码需写在到 handle() 方法中，任务启动时将开始执行该 Job 对象的 handle() 方法。

### 五、投递任务

使用 QueueFactory::get() 获取消息队列的客户端，并调用 put() 方法投递任务，put() 方法返回投递后的任务 ID。

```php
use Wind\Queue\Queue;
use Wind\Queue\QueueFactory;

$queueFactory = di()->get(QueueFactory::class);
$queueClient = $queueFactory->get();

$job = new NotificationJob('贾君鹏', '你妈妈喊你回家吃饭啦！');
$jobId = $queueClient->put($job);
```

任务投递后，如果消费者进程处于空闲状态就会立即拿到该任务，并执行 handle() 方法来处理该任务。

## 进阶

### 任务的超时限制

如果一个任务运行时间过长，那么任务会堵塞住消费者，导致后续的任务都得不到执行，为了防止这种情况发生，消息队列默认限制每个任务最长可执行 60 秒，超出这个时间任务将因超时而被踢出消费者，并且进入下一轮重试。

如果你的任务预计运行时间较长，可以在 Job 里定义 ttr 属性的值来指定它的最长运行时间。

```php
class NotificationJob extends Job {

    // 将任务的最长可运行时间调整为 10 分钟
    public $ttr = 600;

}
```

任务超时的可能性有很多，如果任务在运行中知道自己并没有卡住、比如在处理多条数据时使用了较长的时间，不希望当前任务因运行缓慢被队列系统判定超时，则可以每隔一个循环调用队列的 touch() 方法来让队列系统重新计算自己的超时时间。

```php
class NotificationJob extends Job {

    public $ttr = 60;
    public $messages;

    /**
     * @param array $messages 多条要发送的消息
     */
    public function __construct($messages=[])
    {
        $this->messages = $messages;
    }

    public function handle()
    {
        foreach ($this->messages as $message) {
            NotificationService::send($message['sendTo'], $message['message']);

            // 重设当前任务的超时计时，touch() 可以调用多次
            $this->touch();
        }
    }

}
```

> 调用 touch() 本身也是有成本的，如果自己的任务运转飞快，没有超时的担忧，则没必要调用 touch()。

### 任务的失败重试

如果一个任务运行时抛出了一个未被捕获的异常，则消费者会认为该任务失败了，该任务会在之后经过几轮重试，经过多轮重试后仍然失败，则消费者会放弃执行该任务，将该任务置于“失败”状态。

#### 最大重试次数

要设置任务的最大重试次数，在 Job 中定义 maxRetries，maxRetries 默认为 2，即任务失败后最多重试两次，如果设置为 0 则代表不重试。

```php
class NotificationJob extends Job {

    // 将任务的最大重试次数调整为 5 次
    public $maxRetries = 5;

}
```

#### 重试间隔

任务失败时默认会在 5 秒后重试，如果想改变任务的重试间隔，有两种方法：

第一种方法是简单的设置重试间隔属性 `retryInterval`，该属性是可以是一个 int 代表每次重试都采用相同的间隔秒数，或者是一个 array 根据重试的次数来使用不同的间隔，如果重试的次数超出了 array 的内元素的数量，则采用最后一个数字作为间隔。

```php
class NotificationJob extends Job {

    public $maxRetries = 5;

    //设置为第 1 次重试间隔 10 秒，第 2 次重试间隔 60 秒，再往后重试都是间隔 120 秒
    public $retryInterval = [10, 60, 120];

}
```

另一种方法是可以在任务内定义一个 `retryPolicy` 方法，该方法分别接收两个参数：已经重试的次数（从 0 开始）、触发重试的异常，然后返回下一次重试的间隔秒数，或者返回 false 代表不要重试，你可以在方法内根据其它情况动态控制重试的间隔策略。

```php
class NotificationJob extends Job {

    public $maxRetries = 5;

    public function retryPolicy($attempts, $ex)
    {
        if ($ex instanceof \RuntimeException) {
            //如果是 RuntimeException 则不进行重试
            return false;
        } else {
            //其它异常根据次数决定重试间隔
            return match($attempts) {
                0 => 10, //首次重试隔 10 秒
                1 => 60, //第二次重试隔 60 秒
                default => 120 //之后重试隔 120 秒
            };
        }
    }

}
```

#### 失败时的处理

当一个任务失败时，如果你没有主动捕获该异常，除了会触发状态为 \Wind\Queue\QueueJobEvent::STATE_ERROR 或 \Wind\Queue\QueueJobEvent::STATE_FAILED 的 \Wind\Queue\QueueJobEvent 事件被系统日志记录外，您不会知道它具体失败的原因，如果想要主动捕获任务失败的原因，您可以在 handle() 中套一个大的 try..catch，为了不影响任务的失败判定需要在 catch 中主动抛出任务。

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
$queueClient->put($job, 60);
```

注意指定延迟 60 秒不代表到了该时间后任务一定会执行，这取决于当时消费者是否有空来处理，如果消费者经常不能及时的处理任务，你可以根据所在主机硬件的配置情况适当的增加消费者的数量。

### 取消任务

如果一个任务尚未执行到，或者处于延迟等待的状态时，可以通过消费者客户端的 delete() 方法来删除该任务。

```php
$job = new NotificationJob('贾君鹏', '到点了，该回家吃饭了。');
$jobId = $queueClient->put($job, 60);

//贾君鹏已经提前到家，不用再喊了
$queueClient->delete($jobId);
```

### 消费者数量

默认情况下，Wind 框架仅会为每个队列启动一个消费者，这显然是不够的，可以根据硬件的资源情况加大消费者的数量。

在队列的配置文件中有两个可选参数 processes 和 concurrent，processes 代表启动多少个消费者进程，concurrent 则指定每个进程里启动多少个消费者，以下是配置为队列启动 2 个消费者进程，每个进程 16 个消费者的例子，消费者的总数是 processes * concurrent = 32。

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

### 定义多个队列

除了 default 队列外，你也可以定义多个队列，指定不同的消费者数量和不同的驱动等。

#### 配置额外的队列

在配置文件 config/queue.php 中增加一个名为 notify 的队列，使用 Beanstalkd 作为驱动，队列存入 wind-queue 的 tube 中。

```php
<?php

return [
    'default' => [
        //...
    ],
    'notify' => [
        'driver' => Wind\Queue\Driver\BeanstalkDriver::class,
        'host' => '192.168.4.2',
        'port' => 11300,
        'tube' => 'wind-queue',
        'reserve_timeout' => null,
        'processes' => 2,
        'concurrent' => 8
    ]
];
```

#### 配置额外的消费者进程

不同的队列必须要运行在不同的进程中，所以我们需要创建一个独立的自定义进程来运行 notify 队列。

创建一个消费者进程 `App\Process\NotifyConsumerProcess`，该进程需继承自 `Wind\Queue\ConsumerProcess`，并且将 queue 属性指定为新的队列名称。

```php
<?php

namespace App\Process;

/**
 * 消息发送专用队列
 *
 * @package App\Process
 */
class NotifyConsumerProcess extends \Wind\Queue\ConsumerProcess
{

    //运行 notify 队列的消费者进程
    protected $queue = 'notify';

}
```

然后将 NotifyConsumerProcess 加入自定义进程 config/process.php 配置中。

```php
<?php

return [
    // ...
    Wind\Queue\ConsumerProcess::class,
    App\Process\NotifyConsumerProcess::class
];

```

#### 投递任务到指定队列

投递任务时需要先获取 notify 队列的客户端，在使用 QueueFactory::get() 方法获取队列客户端时指定第一个参数为队列名称。

```php
use Wind\Queue\Queue;
use Wind\Queue\QueueFactory;

$queueFactory = di()->get(QueueFactory::class);

// 获取 notify 队列的客户端
$queueClient = $queueFactory->get('notify');

$job = new NotificationJob('贾君鹏', '你妈妈喊你回家吃饭啦！');
$jobId = $queueClient->put($job);
```

使用另一个队列一般是出于额外的考虑，假设我们有两类以下的任务：

- 第一种是运行时间较短，但是运行速度非常快，比如发送通知的任务，这类任务可以配置较多的消费者，尽可能的快尽可能的及时处理。
- 第二种种是运行时间较长，会长时间占用消费者，但是量一般不大。比如这个任务是调起 ffmpeg 来处理视频或图片，单台服务器的 CPU 资源有限，不能同时启动太多的 ffmpeg，所以希望将这类任务的消费者数量控制在能接受的范围之内。

这两种队列显然是不适合放在一起的，因为第二种任务可能会长时间占用消费者，导致第一种任务得不到及时的处理。或者第二种任务放在第一种队列里运行，无法很好的控制它同时处理的数量。在这种情况下配置两种不同消费者数量的队列，将两种任务放在不同的队列中就可以很好的解决这类问题。

## 队列驱动

### MySQL 驱动
### Redis 驱动
### Beanstalkd 驱动

