# 请求与响应

## 请求

## 响应

响应定义控制器如何向客户端发送数据。

### 简单响应

简单响应即控制器直接返回数据。

#### 文本

在控制器方法中直接返回字符串会以原样输出。

```php

class IndexController {

    public function output()
    {
        return 'Hello World';
    }

    public function simplePage()
    {
        return <<<HTML
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Wind 框架</title>
</head>
<body>
  <p>Hello World</p>
</body>
</html>
HTML;
    }

}

```

#### JSON

返回数组或对象，将会以 JSON 格式输出。

```php
class IndexController {

    public function arr()
    {
        return [
            'name' => 'wind'
        ];
    }

    public function simplePage()
    {
        $obj = new \stdObject;
        $obj->name = 'wind';
        return $obj;
    }

}
```

访问时得到输出

```json
{"name":"wind"}
```

JSON 由 PHP 内置函数 `json_encode()` 进行序列化，如果想配置 JSON 序列化的参数，修改 `config/server.php` 中的 `json_options`。

默认值为：

```php
//...
'json_options' => JSON_UNESCAPED_UNICODE | JSON_INVALID_UTF8_SUBSTITUTE
//...
```

### 复杂响应

可以通过返回 `Wind\Web\Response` 实例来控制响应的细节，如状态码、头信息等等。

```php
use Wind\Web\Response;

return new Response(200, 'Hello World', ['Content-Type'=>'text/plain']);
```

#### 一些例子

```php
use Wind\Web\Response;

$response = new Response(200, 'Hello World');

//修改 HTTP Status Code
$response = $response->withStatus(400);

//添加头信息
$resposne = $response->withHeader('X-Power-By', 'Wind-Framework');

//获取 Header 值，获取时名称可以不区分大小写
$contentType = $response->getHeaderLine('Content-Type');

//修改响应
$body = \Wind\Web\Stream::create('Have a nice day');
$response = $response->withBody($body);

return $response;
```

`Wind\Web\Response` 实现了标准的 PSR-7 `Psr\Http\Message\ResponseInterface`，具体方法请参考 [PSR-7: HTTP message interfaces ](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface).

#### 发送文件

除了直接将文件内容读出发送外，Wind 框架也支持类似于 Nginx 的 X-Sendfile (X-Accel-Redirect) 机制，即高效的以流式输出文件，对于大文件并不需要将文件内容全部读入内存，该机制是基于 Workerman 的 [Http 协议发送文件](https://www.workerman.net/doc/workerman/http/response.html#%E5%8F%91%E9%80%81%E6%96%87%E4%BB%B6)实现，使用时只需添加 Header `X-Workerman-Sendfile` 指向文件完整路径即可。

```php
return $response->withHeader('X-Workerman-Sendfile', '/path/to/file');
```

### 流式响应

Wind 框架支持以流的形式向客户端发送数据，所谓的流式响应可以理解为`边发送边接收`的响应，适用于大数据发送和实时推送场景。

流式数据将以 `chunked` 模式发送给客户端，需要使用 `Wind\Web\Stream\IterableStream` 或 `Wind\Web\Stream\ByteStream` 实现。

#### IterableStream

IterableStream 为可迭代对象流，要了解什么是可迭代对象，请访问 [PHP Iterable](https://www.php.net/manual/zh/language.types.iterable.php)，简单来说，可迭代对象是一种不用一开始就准备完整数据，可以源源不断一边生成一边返回的数据的方法。

以下是可迭代对象流的流式响应的例子，该例子每隔 0.2 秒向客户端发送一堆数字，发送 100 次结束。

```php
use Wind\Web\Stream\IterableStream;
use Wind\Web\Response;

$iterable = (function() {
    foreach (range(0, 100) as $num) {
        yield str_repeat((string)$num, 10)."\r\n";
        delay(0.2);
    }
})();

$stream = new IterableStream($iterable);

return Response::chunked($stream);
```

注意这里的 `$iterable` 是由 `(function() {})();` 生成的，即匿名函数加上 `yield` 生成器执行后的结果才是可迭代对象，只传一个匿名函数是不行的。

#### ByteStream

ByteStream 与 IterableStream 的区别是 ByteStream 不通过可迭代对象发送数据，而是调用 `write` 方法。

```php
use Wind\Web\Stream\ByteStream;
use Wind\Web\Response;

$stream = new ByteStream();

async(function() use ($stream) {
    try {
        foreach (range(0, 100) as $num) {
            $stream->write(str_repeat((string)$num, 10)."\r\n");
            delay(0.2);
        }
    } finally {
        $stream->end();
    }
});

return Response::chunked($stream);
```

请注意 ByteStream 并不知道你是否要停止发送，所要需要调用 `end()` 方法结束发送，否则发送完后客户端会始终处于挂起状态。安全起见，请使用 `try..finally` 在 finally 块中结束发送。

为了防止数据流在不使用时一直占用连接的意外情况，ByteStream 有 `idleTimeout` 参数控制流的闲置超时时间，当 ByteStream 流即没有写入也没有读取时，闲置达到指定时间，流将会被停止，在实例化时指定 `idleTimeout` 参数即可，单位为秒，默认是 30 秒，0 则代表不超时。

```php
//设置闲置超时时间为 5 秒
$stream = new ByteStream(idleTimeout: 5);
```

ByteStream 支持检测是否可以继续写入，如果客户端在接收数据的过程中断开了连接，或者缓冲区已满无法发送，则继续往流中写入数据是没有意义的，我们可以在检测到无法写入时及时停止写入，减少资源的浪费。通过调用 `isWritable()` 来判断是否可写入。

```php
$stream = new ByteStream();

async(function() use ($stream) {
    try {
        foreach (range(0, 100) as $num) {
            if ($stream->isWritable()) {
                $stream->write(str_repeat((string)$num, 10)."\r\n");
                delay(0.02);
            } else {
                //如果连接已经断开则停止写入
                break;
            }
        }
    } finally {
        $stream->end();
    }
});

return Response::chunked($stream);
```

`ByteStream` 有更大的自由度和细节控制，如果不想关心这些细节，建议使用 `IterableStream`。

### Server-Sent Events

Server-Sent Events （简称 SSE） 可以实现服务器实时的多次向客户端推送消息，想详细了解 SSE 请访问[服务器发送事件](https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events)。

在 Wind 框架中，使用 `Wind\Web\ServerSentEventsResponse` 实现 SSE。

```php
use Wind\Web\Stream\ByteStream;
use Wind\Web\ServerSentEventsResponse;

$response = new ServerSentEventsResponse();

async(function() use ($response) {
    try {
        for ($i = 0; $i < 10; $i++) {
            $response->send("Hello World\n$i", id: uniqid());
            delay(0.5);
        }
    } finally {
        $response->end();
    }
});

return $response;
```

`ServerSentEventsResponse` 是基于 `ByteStream` 实现的 `Response` 封装，所以与 ByteStream 类似，也同样有 `idleTimeout` 方法控制超时关断，`isWritable()` 和 `end()` 方法，有必要时需调用 `end()` 关闭连接等。同时也可以调用 Response 的方法设置 HTTP 响应的头信息等等。

```php
$response = new ServerSentEventsResponse(headers: ['X-Power-By'=>'Wind-Framework'], idleTimeout: 10);

$response = $response->addHeader('X-Hello', 'Have a nice day');

async(function() use ($response) {
    try {
        for ($i = 0; $i < 10; $i++) {
            if ($response->isWritable()) {
                $response->send("Hello World\n$i", id: uniqid());
                delay(0.5);
            } else {
                //如果连接已经断开则停止写入
                break;
            }
        }
    } finally {
        $response->end();
    }
});

return $response;
```
