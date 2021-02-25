# 依赖注入 <!-- {docsify-ignore} -->

依赖注入是依赖解耦的一个非常有意义的方式。配合容器用来实现全局的单例对象管理、非单例的对象依赖实例化等等。
所以首先要了解容器与依赖注入的概念。

Wind 框架通过 `php-di` 管理依赖，可以通过 Web 控制器方法参数名称命中、类型命中注入，或使用全局容器实例创建与获取实例。

## 容器

本质上来讲，容器就是一个对象，这个对象存储着应用程序所需的各种对象、变量属性等等以供需要时使用。
在框架中容器最主要用于全局单例对象的管理。Wind 框架通过 php-di 创建了一个全局的单例容器。

可以通过以下方式获取该容器的实例：

### 1. 通过依赖注入获取

如在对应的类，或控制器方法中通过 PSR 16 的 ContainerInterface 反射获取 Container（容器）实例。

```php
<?php
namespace App\Controller;

use \Psr\Container\ContainerInterface;

class IndexController {

	public function actionIndex(ContainerInterface $container) {
	    //$container 即为容器实例
	}

}

```

### 2. 通过 Application 实例获取

应用程序在启动后会将容器保存于应用程序本身的单例中，通过以下方式获取即可。
```php
<?php
use Wind\Base\Application;
Application::getInstance()->container;
```

### 3. 通过 di() 函数

与 2 相同，di() 只是一个快捷获取的函数
```php
<?php
$container = di();
```

## 依赖

光是拿到容器实例并没有什么意义，重要的还是通过容器实例拿到需要的对象。

通过依赖的配置、我们可以做到使用 Interface 获取到对应的实现，让接口和实现解耦。
如缓存的实现可随时切换为 memcache 或 redis 的实现，而不用修改业务代码。

目前支持以下几种方式获取依赖：

### 1. 在构造函数或通过容器调用的方法中使用类型命中获取

目前控制器与由容器实例化的类构造函数支持依赖注入。

以下是在控制器中获取视图实例、请求对象、容器的演示。

```php
<?php

namespace App\Controller;

class IndexController {

	public function actionIndex(Request $request, ViewInterface $view, ContainerInterface $container)
	{

	}

}

```

### 2. 使用 get() 获取
### 3. 使用 make() 获取

### 配置依赖

依赖的配置通过 defintions.php 文件来管理，在配置文件目录下建议 defintions.php 文件，并加入以下内容：

```php
<?php
use function DI\{create, autowire, factory};
return [
	\Wind\View\ViewInterface::class => create(\Wind\View\Twig::class),
    \Psr\SimpleCache\CacheInterface::class => autowire(\Wind\Cache\RedisCache::class),
    //全局 HttpClient
    \Amp\Http\Client\HttpClient::class => factory(function() {
        return \Amp\Http\Client\HttpClientBuilder::buildDefault();
    })
];

```

#### Interface 映射

#### 工厂函数

#### 直接