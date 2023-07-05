# 日志

Wind 框架的日志系统基于 [MonoLog](http://seldaek.github.io/monolog/) 实现，您只需简单的配置即可使用。

## 上手

使用日志时可以直接通过 LogFactory 的 get() 方法获取到 Logger 实例后调用对应方法写入日志。

LogFactory 可以通过[依赖注入容器](/zh-cn/core/di.md )直接取到，以下是一个在 Web 控制器中写入日志的简单例子：

```php
<?php

namespace App\Controller;

use Wind\Log\LogFactory;

class IndexController extends \Wind\Web\Controller {

    public function index(LogFactory $logFactory) {
        $logger = $logFactory->get();
        $logger->info('This is text write by index.');
    }

    public function getLoggerByDi()
    {
        $logFactory = di()->get(LogFactory::class);
        $logger = $logFactory->get();
        $logger->info('This is text write by getLoggerByDi.');
    }

}

```

日志写入后可以在项目的 runtime/log/app.log 中看到如下内容：

```
[2023-03-18 00:00:00 330] app.INFO: This is text write by index. [] []
[2023-03-18 00:00:05 567] app.INFO: This is text write by getLoggerByDi. [] []
```

## 配置

如果想要为不同的日志写入到不同的位置，或者定义实际写入的日志等级、文件保留时长等，则您需要通过配置文件来定义多个日志组，然后为每个组定义独立的处理器（Handler）。

以下是示例项目日志配置文件 config/log.php 的内容：

```php
<?php

use Monolog\Formatter\LineFormatter;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;
use Wind\Log\StdoutHandler;

return [
    //默认日志组
    'default' => [
        // 多个 handlers
        'handlers' => [
            [
                'class' => StreamHandler::class,
                'args' => [
                    'stream' => RUNTIME_DIR.'/log/app.log',
                    'level' => Logger::INFO
                ]
            ],
            [
                'class' => StdoutHandler::class,
                'args' => [
                    'level' => Logger::INFO
                ],
                //为 handler 设置独立的 formatter
                'formatter' => [
                    'class' => LineFormatter::class,
                    'args' => [
                        'dateFormat' => 'Y-m-d H:i:s',
                        'allowInlineLineBreaks' => true
                    ]
                ]
            ]
        ],
        //整个组默认的 formatter
        'formatter' => [
            'class' => LineFormatter::class,
            'args' => [
                'dateFormat' => 'Y-m-d H:i:s v',
                'allowInlineLineBreaks' => true
            ]
        ]
    ]
];

```

可以在日志文件中配置多个日志组，如图中配置了 default 日志组是框架默认的日志组，组中定义了多个日志处理器 (Handler), 有 StreamHandler (写入到文件) 和 StdoutHandler （输出日志到控制台）两个日志处理器。

Handler 的配置结构为一个数组，class 指定 Handler 的类名，args 指定 Handler 初始化的各个具名参数。

还可以使用 formatter 单独指定该 Handler 的格式处理器，也可以直接放在组的子级，作为整个组的格式处理器。

您可以使用不同的 Handler 来定义日志的写入方式，如发送到专用的日志服务器，或者通过 redis, syslogd 等写入日志，关于各个 Handler 和 Formatter 的处理程序可以查阅此文档：[Handlers, Formatters and Processors](http://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html)。

## 使用其它日志组

您可以在配置文件中添加独立的组，来让不同的日志写入到自己的独立日志文件中。

### 配置组

以下示例中我们添加了一个名为 job 的日志组，它使用 RotatingFileHandler 将日志按日期写入到如 job-2023-03-18.log 的文件中，写入日志设置为 INFO 及以上等级，最多保留最近 15 天的日志文件。

```php
<?php

use Monolog\Formatter\LineFormatter;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Logger;

$commonFormatter = [
    'class' => LineFormatter::class,
    'args' => [
        'dateFormat' => 'Y-m-d H:i:s v',
        'allowInlineLineBreaks' => true
    ]
];

return [
    //默认日志组
    'default' => [
        // ... default 日志组的内容
    ],
    //新增 job 日志组
    'job' => [
        'handler' => [
            'class' => RotatingFileHandler::class,
            'args' => [
                'filename' => RUNTIME_DIR . '/log/job.log',
                'level' => Logger::INFO,
                'maxFiles' => 15
            ]
        ],
        'formatter' => $formatter
    ]
];

```

注意这里我们使用了 handler 来配置日志处理程序，而非 handlers，两者的区别是 handler 用于只配置一个日志处理器，而 handlers 可以配置多个。

### 获取组的 Logger 实例

在配置好日志组后，通过 LogFactory 的 get() 方法获取该组的 Logger 实例，通过传递第二个 group 参数指定组名。

```php
$logger = $logFactory->get('app', 'job');
$logger->info('This log write to job group.');
```

日志文件 runtime/log/job-2023-03-18.log

```
[2023-03-18 00:00:00 330] app.INFO: This log write to job group. [] []
```

## LogFactory->get() 的参数

**LogFactory::get(string $name='app', string $group='default');**

**$name** 写入的内容在日志文件的中的前缀名称，默认为 'app'，您可以结合日志关联的业务给定一个名称。

如这是一个记录用户登录来源的日志：

```php
$userId = 1;
$userLogger = $logFactory->get('user-'.$userId);
$userLogger->info('User logged from web.');
```

日志内容：

```
[2023-03-18 00:00:00 330] user-1.INFO: User logged from web. [] []
```

这样我们在查找日志时使用 `group 'user\-1' app.log` 命令便可以方便的查找到关于该用户的相关行为。

**$group** 要获取的 Logger 所使用的配置文件中的日志组，默认为 'default'，指定组名后可以将日志按该组定义的位置和格式写入。

## Logger 的写入方法

Logger 实际上就是 MonoLog 的 Logger 实例，实例提供了 emergency, alert, critical, error, warning, notice, debug 方法来写入不同等级的日志。除了第一个参数 `string $message` 外，还提供了第二个参数 `array $context=[]` 来写入一些关联的数据，默认情况下关联数据会以 JSON 的形式附加在日志内容的后面。

您也可以直接使用 log() 方法，手动传入一个日志等级，以下两种写入方式效果是一样的：

```php
use Monolog\Logger;

$logger = $logFactory->get();

$context = ['categoryName'=>'Test'];
$logger->error('Category create failed cause limited.', $context);
$logger->log(Logger::ERROR, 'Category create failed cause limited', $context);
```

```
[2023-03-18 00:00:00 330] app.ERROR: Category create failed cause limited. {"categoryName": "Test"} []
[2023-03-18 00:00:00 330] app.ERROR: Category create failed cause limited. {"categoryName": "Test"} []
```

## 异步写入

虽然写入一段日志的速度通常非常快，但是由于 PHP 的文件 IO 均是同步写入形式，在同时写入日志量超级多时，日志的写入也有可能影响到业务的并发效率，这种情况下您可以配置日志异步发送到 [TaskWorker](/zh-cn/core/task) 中，TaskWorker 会以多进程队列的形式写入，避免对主要业务进程的影响。


### 使用 TaskWorker 异步写入

使用 TaskWorker 异步模式的前提是您已经配置好 TaskWorker 进程，然后在日志组的 Handler 配置中加入 `async` 选项即可（`async` 为 `true` 时默认为 Task Worker 模式，值也可以是 `\Wind\Log\LogFactory::ASYNC_TASK_WORKER`）。

```php
<?php

return [
    //日志组
    'default' => [
        'handler' => [
            //...
            // 添加 async 选项配置为异步写入
            'async' => true
        ]
    ]
];

```

### 使用独立的进程异步写入

Wind 框架还提供了一种使用独立进程的方式来负责日志的写入，将 async 选项配置为 `\Wind\Log\LogFactory::ASYNC_LOG_WRITER` 代表使用 Log Writer 模式，然后将 `\Wind\Log\LogWriterProcess` 加入到进程配置文件 process.php 中以启用日志写入进程。

**log.php**
```php
<?php

use Wind\Log\LogFactory;

return [
    //日志组
    'default' => [
        'handler' => [
            //...
            // 配置为 LogWriter 写入模式
            'async' => LogFactory::ASYNC_LOG_WRITER
        ]
    ]
];

```

**process.php**
```php
<?php

return [
    // ...
    // 启用日志写入进程
    Wind\Log\LogWriterProcess::class,
];

```

建议您在已经启用 Task Worker 进程的情况下使用 Task Worker 模式，如未启用 Task Worker 则使用 Log Writer 模式更为合适，这样可以更加充分的节省资源。

另一方面考虑的是如果 Task Worker 进程较多，而您的 Logger 也较多，则过多的 Task Worker 进程可能会持有相对较多的日志文件打开句柄，这种情况下使用 Log Writer 则能减少打开文件数。

参考信息：
- MonoLog: http://seldaek.github.io/monolog/

