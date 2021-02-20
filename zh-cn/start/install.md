# 安装 <!-- {docsify-ignore} -->

Wind 框架是以组件化形式管理，想要使用 Wind 框架，只需通过 composer 引入所需的组件即可。

## 运行环境要求

Wind 框架要求运行在 Unix 兼容的系统中，如 Linux，FreeBSD，Mac，或 Windows 10 的 WSL 环境中。

对于 PHP 的要求有：

- PHP 版本 >= 7.2.5
- Pcntl 扩展
- Socket 扩展

## 运行示例

```
git clone https://github.com/wind-framework/example.git wind-example
cd wind-example
composer install
php start.php start
```

## 创建一个 Hello World

我们不使用示例，从零实现一个最简单的输出 Hello World 的 Http 网页服务。

Http 服务器由 wind-framework/web 组件所提供，所以我们只需引入此组件即可。

### 1. 引用依赖

先建立一个目录，在目录中引入依赖。
```
composer require wind-framework/web
```

### 2. 创建应用配置

然后我们在当前目录中建立一个 config 目录用来存放配置文件。

然后在 config 目录中创建以下几个配置文件：
1. config.php（全局基础配置）。
1. components.php (框架加载组件配置)。
1. server.php （启用服务配置）。
1. route.php（路由配置）。

内容分别如下：

#### **config.php**
```php
<?php
//Global Config
return [
    'default_timezone' => 'Asia/Shanghai'
];
```

#### **components.php**
```php
<?php
//要启用的组件列表
return [
    \Wind\Event\Component::class,
    \Wind\Process\Component::class
];

```

#### **server.php**
配置中启动了 2 个工作进程的 http 服务器，绑定到 2345 端口。
```php
<?php

return [
    /**
     * 内置 Server 配置
     */
    'servers' => [
        /**
         * Http 服务器
         */
        [
            'listen' => '0.0.0.0:2345',
            'worker_num' => 2,
            'type' => 'http'
        ]
    ],
    'static_file' => [
        'document_root' => BASE_DIR.'/static',
        'enable_negotiation_cache' => true
    ],
    /**
     * Channel 服务配置
     * addr 默认为 TCP 通讯方式，绑定地址为 127.0.0.1，可指定为 unix:// 通信方式
     * port 为当使用 TCP 通讯方式时使用的端口
     */
    'channel' => [
        'enable' => true,
        'addr' => 'unix:///tmp/wind-'.substr(uniqid(), -8).'.sock',
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
        'worker_num' => 0
    ]
];
```

#### **route.php**

配置中指定了路由 / 指向了一个闭包，该闭包返回了 `Hello World`。

```php
<?php
use \FastRoute\RouteCollector;
return function(RouteCollector $r) {
	$r->addRoute('GET', '/', function() {
        return 'Hello World';
    });
};
```

### 3. 创建启动文件

我们再回到上一层中目录中（程序根目录），在目录中创建一个 start.php 文件，内容如下：
```php
<?php

use Wind\Base\Application;
use Workerman\Worker;

require __DIR__.'/vendor/autoload.php';

define('BASE_DIR', __DIR__);
define('RUNTIME_DIR', __DIR__.'/runtime');

Worker::$pidFile = RUNTIME_DIR.'/wind.pid';

if (!is_dir(RUNTIME_DIR)) {
    mkdir(RUNTIME_DIR, 0755);
}

Application::start();
Worker::runAll();

```

### 4. 运行程序

在程序目录中输入 php start.php start

如果输出以下内容，就代表运行成功了。
```
Workerman[start.php] start in DEBUG mode
--------------------------------------------------- WORKERMAN ---------------------------------------------------
Workerman version:4.0.18          PHP version:7.3.22
---------------------------------------------------- WORKERS ----------------------------------------------------
proto   user            worker                   listen                            processes    status           
unix    username        ChannelServer            unix:///tmp/wind-b8354bdf.sock    1             [OK]            
tcp     username        none                     http://0.0.0.0:2345               2             [OK]                     
-----------------------------------------------------------------------------------------------------------------
Press Ctrl+C to stop. Start success.
```

此时在浏览器输入 http://127.0.0.1:2345/ 将会看到 `Hello World`。

### 5. 常见问题

框架除了 php-fpm 常用的扩展，一般只额外依赖 pcntl 扩展即可，请主要检查 cli 模式下是否包含 pcntl, socket 扩展。