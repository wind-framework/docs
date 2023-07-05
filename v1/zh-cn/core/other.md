# 其它功能组件

## 超时控制

如果想要对一个异步任务进行超时控制，即在指定时间内如果异步的 Promise 没有完成，则抛出超时的异常。

### 普通超时控制

使用 Amp 自带的 timeout 函数即可实现超时控制。以下是5秒钟超时的示例，因为任务实行时长为60秒，所以会得到一个超时异常 `Amp/TimeoutException`。

```php
use function Amp\call;
use function Amp\Promise\timeout;

function longTimeJob() {
    return call(function() {
        echo "Job start\n";
        yield delay(60000);
        echo "Job end\n";
    });
}

yield timeout(longTimeJob(), 5000);
```

注意，超时发生后，原任务并未停止运行，实际上这个 60 秒的任务仍然在执行，只是 timeout() 函数状态已经完成了。真正停止一个运行中的任务实现并不容易实现，如果想要停止一个执行中的任务，请结合 `Amp\CancellationToken` 实现。

### 重新设定超时时间

如果想让一个即将超时的任务重新计算超时时间，比如一个任务原定 5 秒未执行完就会超时，当到第 4 秒时我希望超时时间从 0 开始重新计时，那么 Wind 框架提供了 `TouchableTimeout` 功能，我们称之为可触摸的超时控制，这个超时你只要一触摸它，它的超时时间便会重新计时。

示例：

```php
use Wind\Base\TouchableTimeoutToken;
use function Amp\delay;

$touchable = new TouchableTimeoutToken();

// 4 秒后触摸一下
delay(4000)->onResolve(function() use ($touchable) {
    $touchable->touch();
});

// 9 秒后任务仍未执行完才会超时
yield touchableTimeout(longTimeJob(), 5000, $touchable);
```

触摸超时控制提供了一种类似于延长任务运行时间的机制，当任务知道自己仍然在正常运行，但是原定的时间已经不够的时候，就可以使用此方法推迟超时的时间。

注意 touchableTimeout 与普通超时控制一样，并没有真正的停止掉原任务的运行。
