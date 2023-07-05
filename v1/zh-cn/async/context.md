# 协程上下文

在以 PHP—FPM 为主的传统开发中，通过超全局变量，如 `$_GET`, `$_POST`, `$_COOKIE` 可以方便的获取和存储当前请求中特有的一些信息，基于此可以封装出 `User::id()` 等诸如此类的静态方法来获取当前登录的用户等信息。

但在协程框架中，由于在单个 PHP 进程中会存在多个同时运行的请求或并发逻辑，所以并不能简单的在进程中直接通过全局变量或者静态方法来区分协程各自的信息。以往的做法是在单个协程中，由协程的创建入口传递一个对象（比如 Web 控制器的请求对象），层层传递，一直传递到所需要的地方，以做到单个协程内信息的共享，这个方法的缺点是传递对象太过麻烦，哪怕是中间多个调用的方法函数都不需要这个对象，仅仅是最里面有一个函数需要，整个方法函数栈都需要传递这个对象，成本高昂。

在 Wind 框架中为此提供了协程上下文功能，在单个协程中，只需要通过协程上下文提供的静态方法来存储、获取信息，此信息将只在当前协程有效，并且与其它协程之间相互隔离，所在协程结束后，上下文的信息也自动销毁。自此我们就不需要将协程内的共享信息一路传递，只需简单的将其放进协程上下文中即可。

## Web 组件中的例子

最常见的就是在 Web 页面和接口开发中传递请求对象来获取 GET、POST 和 Header 中的 User-Agent 等信息。在控制器中可以通过 `Request::current()` 静态方法直接获取到当前请求中的请求 Request 对象，我们以获取当前请求的 User-Agent MD5 Hash 为例。

传统做法：

```php
use Wind\Web\Controller;
use Wind\Web\RequestInterface;

class ViewController extends Controller {

    public function myHash(RequestInterface $request)
    {
        return "Current hash is: ".InfoService::getCurrentHash($request);
    }

}

class InfoService {

    public static function getCurrentHash(RequestInterface $request)
    {
        $ua = $request->getHeader('user-agent');
        return md5($ua);
    }

}

```

基于协程上下文的做法：

```php

use Wind\Web\Controller;
use Wind\Web\Request;

class ViewController extends Controller {

    public function myHash()
    {
        return "Current hash is: ".HashService::getCurrentHash();
    }

}

class HashService {

    public static function getCurrentHash()
    {
        $ua = Request::current()->getHeader('user-agent');
        return md5($ua);
    }

}

```

## Context 协程上下文方法

Wind 框架提供了 Context 类来自行存取协程上下文中的数据，使用很简单，最常用的就是 `get`, `set` 方法。

```php
use Wind\Base\Context;
use function Amp\async;
use function Amp\delay;
use function Amp\Future\await;

function justWait() {
    //无需传递参数，便能获取当前协程中的数据
    $id = Context::get('id');
    echo "$id start..\n";

    delay(2);

    $id = Context::get('id');
    echo "$id finished after 2 seconds.\n";
}

$futures = [];

//生成 10 个协程，每个协程都有自己的 id 数据存储在协程上下文中
for ($id=1; $id<=10; $id++) {
    $futures[] = async(function() use ($id) {
        Context::set('id', $id);
        justWait();
    });
}

await($futures);

```

> 要注意的是 Context 只会关联到当前的协程，如果与不在协程中，则数据会被存储到类似于全局的单例中，这种情况下可能会出现内存泄漏的问题。常见的协程环境是 Web 的控制器调用栈，消息队列的 handle() 调用栈。

> 如果自己手动创建了一个协程，比如使用 async() 手动创建了一个协程环境，则这个协程是脱离当前协程的，该协程内无法读写当前协程上下文的数据。

