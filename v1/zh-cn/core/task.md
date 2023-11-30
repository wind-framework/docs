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
channel.enable | 是 | 布尔值 | true | 是否启用 Channel 进程间通讯组件，建议启用，启用后框架将启动一个 Channel 服务进程。
channel.addr | 否 | 字符串 | 127.0.0.1 | Channel 服务监听地址，支持 TCP 和 Unix Domain Socket 两种形式，指定 IP 地址即为监听 TCP 地址，指定 unix:// 协议的文件名即为使用 Unix Domain Socket 协议，建议使用 Unix Domain Socket 协议。
channel.port | 否 | 整型 | 2206 | 当使用 TCP 形式时所监听的端口。
task_worker | 否 | 数组 | | Task Worker 配置
task_worker.worker_num | 否 | 整型 | 0 | Task Worker 的工作进程数量，数量越多则其能同时执行的同步任务数越多，但相应的内存占用也会更多。

## 使用

使用 Task Worker 来执行同步或 CPU 密集型任务很简单，只需要将对应的闭包和参数通过 TaskWorker 发送即可。

### 使用 Task 类

#### \App\MyTask\FooTask 类
```php
<?php

namespace App\MyTask;

class FooTask {

    public static function take($p1, $p2='') {
        $value = md5($p1.$p2);
        //Do something
        return $value;
    }

}
```

#### 调用 Task Worker 执行

```php
$result = \Wind\Task\Task::execute('\App\MyTask\FooTask::take', 'Hello World', 'param 2')->await();
print_r($result);
```

Task 类的 execute 方法第一个参数为要执行的闭包名称，后面跟要传递的参数。

因为 Task Worker 是通过 Channel 使用序列化进程进程通信，所以不能序列化的闭包和值将无法正常发送至 Task Worker 执行。
如实例化类中的动态方法，或者传递、返回了复杂的对象实例等等。

需要注意的是 Task::execute() 方法并不直接返回执行的结果，而是返回了一个 Future() 对象，如果要等待并获得对象，请调用 Future 对象的 await() 方法，或者直接使用 Task::await() 方法。

### Task 的方法

Task 除了 execute() 方法外，还提供了其它适用于不同场景的方法。

#### **Task::execute(callable $callable, ...$args): Future**

执行一个任务，并且获取执行的 Future，获取 Future 之后可以进行自由的并发控制。

#### **Task::await(callable $callable, ...$args): mixed**

执行一个任务，并且获取执行的结果，与 execute() 不同的是此方法会自动等待结果的返回，实际上就是 `Task::execute(...)->await()` 的封装。

#### **Task::submit(callable $callable, ...$args): void**

执行一个任务，即不等待也不关心结果。submit() 的功能是当前业务将 callable 传递给 TaskWorker 执行，但是不等待执行的结果，也不关心执行是否成功，与 execute() 类似，但是 submit() 并不会返回 Future 对象，所以你无无法得知它执行是否完成。这适用于一些对结果漠不关心的场景。

### 使用 compute 函数

`compute()` 函数是 `Task::await()` 的简单封装。

```php
$value = compute('\App\MyTask\FooTask::take', 'Hello World');
var_dump($value);
```

### 传递的执行体

传递的执行体理论上只要是 php 中的 [callable](https://www.php.net/manual/zh/language.types.callable.php) 结构便都能执行。

在 Wind 框架中对传递的执行体做了一些处理，支持传递动态对象（有限制）和闭包等。

### 传函数或静态方法名称

传递一个函数名称，如 md5_file 并获取执行结果。

```php
$fileMd5 = compute('md5_file', '/path/to/file');
```

传静态方法中的类名需传递包含命名空间的完整类名，静态函数调用也支持数组的 callable 结构。

```php
$result = compute('App\Example\Assocate::knock', 'params one string');
$result2 = compute(['App\Example\Assocate', 'knock'], 'params one string');
```

### 传递类实例的方法调用

类的实例方法调用有以下几个重点：

1. 类并不会实例化后传递，而是只传递一个名字，由 Task Worker 实例化后调用方法，所以类的构造方法必须是无参数，或者参数是通过依赖注入传递。
1. 且类中间的的其它状态不会保存，因为 Task Worker 调用前只做实例化而不会调用其它方法，所以不能依赖实例化后的其它状态值。
1. 要调用的方法必须是 public 属性。

以下是正确的示例

```php
namespace App\Example;

use Wind\Log\LogFactory;

class MyClass {

    protected $logger;

    //参数支持依赖注入
    public function __construct(LogFactory $logFactory) {
        $this->logger = $logFactory->get('app');
    }

    //方法是 public 属性
    public function myMethod($id)
    {
        return 'Your id is '.$id;
    }

}

asyncCall(function() {
    $obj = new MyClass;
    $result = compute([$obj, 'myMethod'], 123);\
    echo $result; //输出 'Your id is 123'
});

```

以下是错误示例

```php
namespace App\Example;

class MyClass {

    protected $myName;
    protected $myGroup;

    //不能使用不可依赖注册的参数
    public function __construct($myName) {
        $this->myName = $myName;
    }

    public function setMyGroup($group)
    {
        $this->myGroup = $group;
    }

    //方法不是 public 无法被调用
    protected function myMethod($id)
    {
        return "You are {$this->myName}({$this->myGroup)})";
    }

}

asyncCall(function() {
    $obj = new MyClass('Tommy');

    //不能有中间状态，因为传递到 Task Worker 时并不会执行 setMyGroup
    $obj->setMyGroup('BJ');

    //中间状态传到 Task Worker 也不行，因为每次调用都是单独的
    compute([$obj, 'setMyGroup'], 'BJ');

    //以下直接报错
    $result = compute([$obj, 'myMethod'], 123);
    echo $result;
});

```


## 注意

与 TaskWorker 间是通过 Channel 组件通信的，而消息是经过 serialize 序列化传递，所以传递的消息和返回值不建议过大，比如序列化后可能消息有 1M 以上的就不建议使用此方式，否则传递消息本身会带来较大的时间消耗。比如我们要在 TaskWorker 里进行文件的 md5 值计算，建议传递文件名，由 TaskWorker 来计算它的 md5 并返回，而非传递整个文件内容。

- 另外由于 php 序列化的限制，不建议传递太过复杂的对象，以免丢失数据。
- 支付传递闭包，但不建议使用闭包。
