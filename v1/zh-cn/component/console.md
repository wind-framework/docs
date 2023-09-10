# 命令行

Wind 框架的命令行基于 symfony/console 实现。

## 安装

安装命令行组件

```
composer require wind-framework/console
```

确保配置目录 `config` 中有命令行配置文件 `console.php`，默认内容如下，如果没有该内容请手动创建。

console.php
```
<?php
//Console Config
return [
    'command' => [
        'scan_ns_paths' => [
            'App\\Command' => BASE_DIR.'/app/Command'
        ]
    ]
];

```

配置中的 `command.scan_ns_paths` 配置为扫描命令行所在的命令空间及对应目录配置列表，Wind 框架会从该命令空间对应的目录中扫描所有的命令，如果命令不在该命名空间的目录下，将无法被扫描和使用。

您也可以使用注解标记其它位置的命令，注解标记的方法在本文档之后列出。

## 如何使用

框架在根目录提供了 wind 作为命令行的入口，执行 `php ./wind` 即可使用命令行。

```
$ ./wind
Wind Framework Console

Usage:
  command [options] [arguments]

Options:
  -h, --help            Display help for the given command. When no command is given display help for the list command
  -q, --quiet           Do not output any message
  -V, --version         Display this application version
      --ansi|--no-ansi  Force (or disable --no-ansi) ANSI output
  -n, --no-interaction  Do not ask any interactive question
  -v|vv|vvv, --verbose  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug

Available commands:
  completion       Dump the shell completion script
  help             Display help for a command
  list             List commands
 make
  make:command     Create a console command.
```

命令提供了 help, list 命令用于查看帮助和已有命令列表，使用 `php ./wind help` 或 `php ./wind list` 执行对应命令。

## 创建自定义命令

可以通过内置命令 `make:command` 来创建一个自定义命令，或手动在配置 `console.command.scan_ns_paths` 指定的命名空间目录下创建。

### 使用内置命令创建

内置命令使用帮助如下：

```
$ ./wind help make:command
Description:
  创建一个控制台命令.

Usage:
  make:command <name> <className>

Arguments:
  name                  命令的名称，如 'user:info'
  className             命令的类名，如 'UserInfoCommand'
```

我们在创建一个 `user:info` 命令为例，类名为 `UserInfoCommand`。

```
$ ./wind make:command user:info UserInfoCommand
Create command user:info successfully at ~/wind-app/app/Command/UserInfoCommand.php.
```

看到以上信息就代表命令已经创建成功了，打开创建好的文件，就可以对命令进行编辑，在 `execute()` 方法中加入自己的逻辑代码。

### 手动创建

在 `app/Command` 中手动创建一个继承自 `Symfony\Component\Console\Command\Command` 的类，即可作为一个自定义命令。

通过 `configure()` 方法配置命令的名称，介绍，入参等，在 `execute()` 方法中加入自己的业务逻辑代码。

以下是上一步创建的 `user:info` 命令的示例。

```php
<?php

namespace App\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class UserInfoCommand extends Command
{

    protected function configure()
    {
        $this->setName('user:info')
            ->setDescription('Command description.');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('Hello World');
        return self::SUCCESS;
    }

}

```

### 通过注解标记命令

在 `console.command.scan_ns_paths` 配置的命名空间目录以外创建的命令不会被 Wind 框架扫描到，除了加到命令的命名空间扫描的配置项外，还可以通过注解来标记其它位置的命令，这样命令同样可以被使用。

使用注解的前提是注解的命令类能够被注解扫描到，如我在 `app/External` 中创建了一个带有命令注解的 `App\External\ExternalCommand` 命令类：

```php
<?php

namespace App\External;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Wind\Console\Annotation\Command as AnnotationCommand;

// 打上注解
#[AnnotationCommand]
class ExternalCommand extends Command
{

    protected function configure()
    {
        $this->setName('external:info')
            ->setDescription('My external information.');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('Hello World');
        return self::SUCCESS;
    }

}

```

为了确保 External 命令能被注解扫描到，我们将 `App\External` 命令空间加到注解扫描配置中。

注意这里我们在命名空间后面加了 `@console`，这是因为命令仅用于控制台，在 server 模式中扫描这类注解并无意义，在命名空间后添加 `@console` 代表限制该命名空间的扫描仅在 console 模式中生效，这样可以避免其它模式下的性能浪费。当然这种限制是可选的，你也可以不加。

config.php

```php
<?php
//Global Config
return [
    //...
    'annotation' => [
        'scan_ns_paths' => [
            //...
            'App\\External@console' => BASE_DIR.'/app/External'
        ]
    ],
    //...
];
```

## 配置命令

以我们最常用的命令名称、介绍、输入参数为例，命令的配置大多是在 `configure()` 方法中，但也有些参数有多种配置方式。

### 配置命名名称和描述

在 `configure()` 方法中调用 `$this->setName()` 和 `$this->setDescription()` 来配置命名的名称和描述，配置方法可以链式调用。

```php
class UserInfoCommand extends Command
{

    protected function configure()
    {
        $this->setName('user:info')
            ->setDescription('Show user information.');
    }

}
```

通过 `AsCommand` 注解配置名称和描述。

```php

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;

#[AsCommand('user:info', 'Show user information.')]
class TestCommand extends Command
{
}

```

在构造函数中传入名称（不建议使用这种方法）。

```php
class TestCommand extends Command
{

    public function __construct(string $name = 'user:info')
    {
        parent::__construct($name);
    }

}
```

### 配置输入参数

通过 `addArgument()` 方法，可以定义命令行的输入参数和选项。

`addArguments(string $name, int $mode = null, string $description = '', mixed $default = null)` 方法支持参数：

- **$name** 参数的名称。
- **$mode** 参数模式，InputArgument::REQUIRED 为必须的参数，InputArgument::OPTIONAL 为可选参数，为必须的参数，InputArgument::IS_ARRAY 则代表是一个任意多值列表，多值列表的参数必须是最后一个参数。
- **$description** 参数的描述
- **$default** 可选参数的默认值。

在 `execute()` 方法中通过 `InputInterface` 的 `getArgument()` 方法来获取对应名称参数的值。

```php
use Symfony\Component\Console\Input\InputArgument;

class UserInfoCommand extends Command
{

    protected function configure()
    {
        $this->setName('user:info')
            ->setDescription('Show user information.')
            ->addArgument('userId', InputArgument::REQUIRED)
            ->addArgument('parts', InputArgument::OPTIONAL, 'User info parts, eg: nickname, birthday, or all?', 'all');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $userId = $input->getArgument('userId');
        $parts = $input->getArgument('parts');
        return 0;
    }

}
```

执行 help 后可以看到该命令的输入参数帮助

```
$ ./wind help user:info
Description:
  Show user information.

Usage:
  user:info <userId> [<parts>]

Arguments:
  userId
  parts                 User info parts, eg: nickname, birthday, or all? [default: "all"]

...
```

## 输出

在 `execute()` 方法中使用 `OutputInterface` 的 `write()` 或 `writeln()` 方法来输出。

```php
class UserInfoCommand extends Command
{

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->write('Hello ');
        $output->write(' World.');
        $output->writeln('Have a nice day.');
        return 0;
    }

}
```

## 返回值

返回值可以向终端反馈命令运行结果的状态，`execute()` 方法的返回值定义是 `0 代表运行结果正常，否则返回错误码`，错误码的定义参照 https://tldp.org/LDP/abs/html/exitcodes.html

建议直接使用内置的常量来代替返回码：

```php
class UserInfoCommand extends Command
{

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        switch ($input->getArgument('condition')) {
            case 'ok':
                //正常
                return self::SUCCESS;
            case 'error':
                //失败
                return self::FAILURE;
            default:
                //不合法
                return self::INVALID;
        }
    }

}
```

## 其它

注意在命令行环境下不可以使用与 Workerman 环境有关的组件，如 Worker 进程、AsyncTcpConnection 连接等。事实上 Workerman 对于本框架的作用是提供进程模型以及服务程序，这些都是在 server 模式下有效。

在命令行状态下，可以自由使用本框架内的任意组件，对应的协程环境均有效。如要使用异步/协程的方式建立 TCP 连接，请使用 [amphp/socket](https://amphp.org/socket)。

了解更多命令行的使用方法，参见：http://www.symfonychina.com/doc/current/components/console.html

