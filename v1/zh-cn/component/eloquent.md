# Eloquent

Wind 框架提供了基于 Laravel 数据库组件 Eloquent 的封装，结合了 amphp/mysql 协程客户端的异步并发性能和 Eloquent 的易用性。

## 安装
```
composer require wind-framework/eloquent
```

将 `\Wind\Eloquent\Component::class` 加入到配置文件 `config/components.php` 中。

确保配置文件 `config/database.php` 中有正确的数据库配置，该配置与 Wind 框架的数据库配置一致。

现在开始您可以使用同 Laravel 一样的的数据库查询，查询构造器，以及 Model了。

## 使用

```php
// use DB Facade
use Illuminate\Support\Facades\DB

$user = DB::table('users')->first();
print_r($user);

// define a user model, and query by model
class User extends \Illuminate\Database\Eloquent\Model {

    protected $table = 'users';

    public function books()
    {
        return $this->hasMany(Books::class)
    }

}

$user = User::query()->with('books')->first();
print_r($user?->toArray());
```
