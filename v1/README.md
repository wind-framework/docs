# Wind Framework 1.0

> An awesome project.

## 介绍

Wind Framework 是一款纯 PHP（无第三方扩展依赖）的高性能协程框架。

近年来 PHP 的发展，无论是 PHP 7 的优化，还是 PHP 8 加入的 JIT，PHP 的性能都有了长足的进步，但对于整个语言的生态来说并没有带来任何进步，PHP 当前生态的瓶颈在于运行模式的瓶颈，随着技术的变化趋势，大家发现对于服务端程序，目前绝大部分的瓶颈是在 IO 的部分。而异步和协程化才会让 PHP 有质的变化。

异步和协程不仅让 PHP 程序的性能有了巨大的提升，还可以使 PHP 的功能性做到完全的自足。

传统的 php-fpm 做法，PHP 的应用场景非常有限，很多功能需要依赖周边工具做到，比如计划任务通过 crontab 来设置，消息队列可能以计划任务每分钟启动来执行，或通过进程的守护 Supervisord 来做一个很拙劣的长驻，基于对于数据库之类的连接数暴涨也要使用一些中间件，还有很多场景甚至是束手无策或者实现非常之差，php-fpm 碰到高并发时，实际并发数受到进程数的限制，想要把并发数做大实际付出也非常之大，所以往往企业规模做大，或者业务场景复杂之后都要引入其它语言的方案，这表面上是因为其它语言的生态问题，核心还是因为其它语言支持多线程或协程这两个重要的特性。

而基于纯 PHP 的协程框架，PHP 可以用相对非常少的资源实现以上的这些功能。

Wind 框架的特点是协程框架，这主要依赖于两个核心组件，分别是 Workerman 和 Amphp，Workerman 提供了异步 Socket 组件、多进程的架构，Amphp 提供了纯 PHP 的协程实现，及大量的协程生态。

## 纯 PHP 协程

Wind Framework 的纯 PHP 的协程基于 PHP 的一个重要概念 Fiber 而来，通过 Fiber, PHP 可以做到从执行栈的任意层级和任意位置进行暂停、出让和恢复，从而做到将异步的程序协程化。

关于 Fiber 的原始概念 Generate/yield 相关描述可以参照《[在PHP中使用协程实现多任务调度](https://www.laruence.com/2015/05/28/3038.html)》，原文《[Cooperative multitasking using coroutines (in PHP!) ](https://www.npopov.com/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html)》。

了解 JavaScript ES7 的朋友们应该知道 Promise 及 `await/async`，在 JavaScript 中使用协程需要使用 `async` 在相关的作用域中创建出协程环境，然后在协程环境中使用 `await` 来同步等待执行结果。

```javascript
async function getResult() {
    const res = await fetch('/example');
    const data = await res.json();
    return data.something;
}
```

在本框架的 v0 版本中也是类似的概念，我们需要使用 call() 来代替 async 创造出协程环境，使用 yield 来代替 await 等待结果。但在 v1 版本中有着巨大的变化，基于 Fiber 的 Amphp 提供了隐式协程，我们在开发中不需要过多的关心协程和异步底层的概念，协程也不会对调用栈造成污染，一切看起来与以前的同步开发方式无异。

```php
function getResult() {
    $result = $this->request('/example')
    $buffer = $result->getBody()->buffer();
    $data = json_decode($buffer, true);
    return $data['something'];
}
```

在 PHP 中，协程仍可以返回传统的 Future 概念对象，但 Future 提供的 await() 方法可以暂停整个调用栈，也无需对函数做出如 async 样的特殊标记，使用纯 PHP 的协程将易如反掌。

## 功能

基于卓越的多进程管理、异步 Socket 和协程功能，框架实现了以下组件：

- 目前框架拥有以下组件：
- HTTP 服务器（支持基于控制器路由的动态程序和静态文件）
- 依赖注入
- 缓存（实现 PSR-16 SimpleCache 的协程缓存）
- 进程信息收集组件
- 定时任务组件
- 协程 MySQL 客户端、支持连接池、查询构造器
- 日志组件（基于 MonoLog，支持异步写入）
- 自定义进程组件
- 异步消息队列组件（支持 Redis、Beanstalk 作为驱动）
- 协程 Redis 客户端
- TaskWorker（可将同步调用发到其它进程为异步调用）
- 视图组件（支持 Twig 等多种实现）

让我们一起释放 PHP 的潜力。
