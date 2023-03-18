# 路由

Wind 框架支持非常灵活的路由配置，支持路由分组、命名空间、中间件配置，以及路由配置嵌套等。

路由配置文件位于 config/route.php 中。

## 配置格式

让我们先来看一个演示的路由配置示例：

```php
<?php
/**
 * Router Config
 *
 * groups[]
 * - namespace
 * - prefix
 * - middlewares
 * - routes
 *    - name
 *    - middlewares
 *    - handler
 */
return [
    //app group
    [
        'namespace' => 'App\Controller',
        'routes' => [
            'get /' => 'IndexController::index',
            'get /gc-status' => 'IndexController::gcStatus',
            'get /gc-recycle' => 'IndexController::gcRecycle',
            'get /cache' => 'IndexController::cache',
            'get /soul' => 'DbController::soul',
            'get /soul/{id:\d+}' => 'DbController::soulFind',
            'get /db/concurrent' => 'DbController::concurrent',
            'get /sleep' => 'IndexController::sleep',
            'get /block' => 'IndexController::block',
            'get /exception' => 'IndexController::exception',
            'get /phpinfo' => 'IndexController::phpinfo',
        ],
        'groups' => [
            //test group (group can also have a key name)
            'g1' => [
                'prefix' => '/test',
                'middlewares' => [\App\Middleware\TestMiddleware::class],
                'routes' => [
                    'get task' => 'TestController::taskCall',
                    'get client-ip' => [
                        'name' => 'test.cache',
                        'middlewares' => [\App\Middleware\TestMiddleware::class],
                        'handler' => 'TestController::clientIp'
                    ],
                    'get|post closure' => function(\Workerman\Protocols\Http\Request $req) {
                        return $req->uri();
                    },
                    'get queue' => 'TestController::queue',
                    'get http' => 'TestController::http',
                    'get log' => 'TestController::log',
                    'get file' => 'TestController::sendFile',
                    'post upload' => 'TestController::uploadFile'
                ]
            ],
            //g2 group
            [
                'prefix' => 'g2'
            ]
        ]
    ],

    //static group
    [
        'namespace' => 'Wind\Web',
        'routes' => [
            'get /static/{filename:.+}' => 'FileServer::sendStatic',
            'get /{filename:favicon\.ico}' => 'FileServer::sendStatic'
        ]
    ]
];
```

路由的配置由一系列分组组成，关键字包含 namespace, prefix, middlewares, routes, groups, 并且都是可选的。

各关键字的含义见下列内容。

## groups - 分组

路由本身是由一系列分组组成，从顶层开始相当于 groups 配置，groups 下包含多个组，每个组都可以有一套自己的配置，
该组的配置会一直继承到组下的每一个子组和路由。

groups 包含组，每一个组又可以包含多个子组，如：
```php
<?php
//最顶层就相当于 groups
return [
    //组1
    [
        //组的配置，都是可选
        'namespaces' => 'App\Controller',
        'routes' => [
            //...
        ],
        //组本身还可以再包含子组，子组也可以继续包含子组，支持无限嵌套
        'groups' => [
            //子组1
            [],
            //子组2
        ]
    ],

    //组2，可以给组一个键名，目前并无作用
    'api' => [
        'middlewares' => [],
        'routes' => []
    ],

    //组3、组4 等等
    //...
];
```

组的最大作用就是配置的继承，每一个子组都会继承往上每一层组的除 routes 以外的所有配置。

## namespace - 命名空间

命名空间的配置可以免去组下每一个路由都要写完整的类名的烦恼。

一个简单的例子：
```php
<?php
return [
    [
        'routes' => [
            'get /' => 'App\Controllers\IndexController::index',
            'get /about' => 'App\Controller\AboutController::index'
        ]
    ]
];
```

在上面的例子中，两条路由均是 App\Controllers 命名空间下的路由，当我们使用了 `namespace` 的配置后，
组下的每一条路由的控制器都会继承该命名空间，并且命名空间也支持嵌套。

如下：

```php
<?php
return [
    [
        'namespace' => 'App\Controllers',
        'routes' => [
            'get /' => 'IndexController::index',
            'get /about' => 'AboutController::index'
        ],
        'groups' => [
            [
                //命名空间也可以继承
                'namespace' => 'Test',
                'routes' => [
                    'get /test/' => 'YesController::index' //相当于 App\Controllers\Test\YesController::index
                ]
            ]
        ]
    ]
];
```

## prefix - 路径前缀

前缀的作用是可以省去组下相同前缀路由路径的反复编写。

继续拿两个例子对比：

```php
<?php
return [
    [
        'namespace' => 'App\Controllers\User',
        'routes' => [
            'get /user/articles' => 'ArticleController::index',
            'get /user/comments' => 'CommentController::index'
        ]
    ]
];
```

例子中两条路由表 `/user/articles` 和 `/user/comments` 均拥有相同的 `/user` 前缀，使用 prefix 配置后便可省去写相同的前缀部分，且前缀配置同会支持组的继承。

```php
<?php
return [
    [
        'namespace' => 'App\Controllers\User',
        'prefix' => '/user',
        'routes' => [
            'get /articles' => 'ArticleController::index', //get /usr/articles
            'get /comments' => 'CommentController::index' //get /usr/comments
        ],
        'groups' => [
            [
                'prefix' => '/info',
                'routes' => [
                    'get /avatar' => 'InfoController::avatar', //get /user/info/avatar
                    'get /password' => 'InfoController::password' //get /user/info/password
                ]
            ]
        ]
    ]
];
```

## middlewares - 中间件

中间件是 Web 服务扩展性的重要部分，中间件的使用见【[中间件](/zh-cn/component/middlewares.md)】一章，路由表中同样也支持中间件的配置。

middlewares 关键字的值是一个数组，数组中是一系列中间件的类名，而且同样支持分组继承，需要注意的是子组中如有相同的中间件系统并不会排重，
此时访问该路由时中间件会被调用多次，所以需要注意中间件的配置及顺序。

以下示例中第一个组使用了 AccessLogMiddleware 中间件， 而名为 info 的子组中会同时应用 AccessLogMiddleware, JsonMiddleware, TestMiddleware 三个中间件：

```php
<?php
return [
    [
        'namespace' => 'App\Controllers\User',
        'prefix' => '/user',
        'middlewares' => [\Wind\Web\Middleware\AccessLogMiddleware::class],
        'routes' => [
            'get /articles' => 'ArticleController::index',
            'get /comments' => 'CommentController::index'
        ],
        'groups' => [
            'info' => [
                'prefix' => '/info',
                'middlewares' => [
                    \Wind\Web\Middleware\JsonMiddleware::class,
                    \App\Middleware\TestMiddleware::class
                ],
                'routes' => [
                    'get /avatar' => 'InfoController::avatar', //相当于 /user/info/avatar
                    'get /password' => 'InfoController::password' // /user/info/password
                ]
            ]
        ]
    ]
];
```

## routes - 路由表

路由的配置最重要的是路由表，没有路由表，以上的一切都没有意义。

路由表的值由键值对的数组组成，格式为：
```
[
    '请求方式 路径' => '控制器方法或可调用结构',
    ...
]
```

其中键的格式为 请求方式(Http Method)+空格+路径，

### Http Method

目前支持定义的请求方式有：GET, POST, PUT, DELETE, PATCH，其它的请求方式理论上也支持，但不建议直接使用。

如果一个路由支持多个请求方式，可以使用符号 `|` 来分隔，如 `get|post|put` 代表该路由同时支持 GET, POST, PUT 方式的访问。

请求方式大小写不敏感。

### 路径及参数

Wind 框架路由的底层基于 [nikic/FastRoute](https://github.com/nikic/FastRoute) 实现，所以支持相同的参数配置。

在路径中使用大括号 `{}` 包裹的名称即代表路由参数，参数将作为相同名称的调用参数传递给控制器的方法中。

如 `'get /user/{id}' => 'UserController::view'` 即当请求 `/usr/100` 时将访问 `UserController::view` 方法，并传递一个名为 id 的参数。

```php
<?php

class UserController extends Controller {

    public function view($id) {
        return $id; //$id='100'
    }

```

上面的例子中并没有限制 `{id}` 参数的格式，如果用户访问 `/usr/wind` 此时 `$id` 的值就是 `'wind'`，如果要限制参数的格式，
在参数名称后跟上 `:正则表达式` 来做限制，比如 id 要限制为纯数字，使用 `{id:\d+}` 即可。

一些例子：
- `{id:\d+}` id=纯数字
- `{id:\w{5,10}}` id=5-10位之间的字母和数字
- `{act:(view|modify)}` act 只能是 view 或 modify 之一

路由中可以同时有多个参数，如 `get /posts/{type:(article|comment)}/{number:\d+}`

### 路由方法

在以上的大部分例子中路由都类似于 `'get /' => 'IndexController::index'`，这是一个 [PHP 的 Callable 结构](https://www.php.net/manual/zh/language.types.callable.php)，代表当我们访问 / 路径时，
实际上请求会发送至 IndexController 类的 index 方法。

路由所指向的方法同样支持多种结构，除常见的 `Class::Method` 以外，还支持闭包，如 `function($id) { ... }`，
或者指向一个函数的名称，如 `'my_hello_function'`。

路由的方法还支持以数组的形式定义一个更复杂的配置，如为单独一个路由指定中间件和名称等。

一些例子：

```php
<?php
return [
    [
        'routes' => [
            'get /' => 'App\Controllers\IndexController::index',
            'get /view/{id}' => function($id) {
                return "Id is $id";
            },
            'get /hello' => 'my_hello_function',
            'post /complex' => [
                'name' => 'router_name',
                'middlewares' => [], //router own middlewares
                'handler' => 'SomeController::action'
            ]
        ]
    ]
];
```

#### 一些细节

需要注意的是对于 `Class::Method` 结构，Wind 框架会自动判断对应的方法是静态方法或是动态方法，如果是动态方法，则会实例化该类后动态调用，
如果是静态方法则会直接静态方式调用。

路由指向的方法、函数都默认支持协程环境，我们无需使用 call() 函数进行包裹。
