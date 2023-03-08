# 数据模型

Wind 框架提供了一种相对简单的数据模型来实现便捷的数据表映射操作。

## 创建模型

新建一个类继承 Wind\Db\Model 即可创建一个模型，如下即可创建一个 User 模型。

```php
<?php

namespace App\Models;

use Wind\Db\Model;

/**
 * User Model
 */
class User extends Model {

}

```

## 配置模型表名

默认情况下，模型会根据模型类来得出表名，如 App\Models\User 模型便会得出 `user` 表名，如果想自定义表名，在模型中定义常量 TABLE 指定表名即可。

```php
/**
 * User Model
 */
class User extends Model {

    const TABLE = 'users';

}

```

## 配置所属数据库连接

如果想要指定模型所要使用的数据库连接，在模型中定义 CONNECTION 常量，指定 config/database.php 中配置的名称即可。

假设 database.php 中有以下配置
```php
<?php

return [
    'default' => [
        'type' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', 3306),
        'database' => env('DB_DATABASE', 'test'),
        'username' => env('DB_USER', 'root'),
        'password' => env('DB_PASS', ''),
        'prefix' => env('DB_PREFIX', ''),
        'charset' => env('DB_CHARSET', 'utf8mb4'),
        'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
        'pool' => [
            'max_connections' => 30,
            'max_idle_time' => 90
        ]
    ],
    'data_store' => [
        //... data_store 连接的配置
    ]
];
```

此时我们要指定 User 模型使用 data_store 的连接。

```php
/**
 * User Model
 */
class User extends Model {

    const CONNECTION = 'data_store';

}

```

## 配置主键

模型会默认使用 `id` 作为模型的主键， 模型对单条数据的 find、保存、修改均会基于主键进行操作，当查询出一条模型时，确保其中包含主键的值，否则操作将失败，如果想要指定主键的字段名，在模型中定义常量 PRIMARY_KEY。

```php
/**
 * User Model
 */
class User extends Model {

    const PRIMARY_KEY = 'uid';

}

```

## 数字值自增和自减

如果想要对模型的某个字段进行自增和自减的操作，比如给文章的浏览数 +1，应该使用模型提供的 increment, decrement 方法来递增和递减。

```php
<?php

use App\Models\Article;

// 这相当于 UPDATE `article` SET `views`=`views`+1 WHERE `id`=1
$article = yield Article::find(1);
yield $article->increment('views');
```

以下是错误的示例，因为在拿到 Article 实例后，views 可能已经被其它位置更新了，这样会导致本实例 views 的值 +1 后覆盖数据库已经被更新的值。

```php
<?php

use App\Models\Article;

// 错误示例，此方法存在并发问题
$article = yield Article::find(1);
$article->views += 1;
yield $article->update();
```

### 一次递增递减大于1的值
如果要递增或递减的数据不只是 1，可以传递第二个参数指定一个数字，以下是对商品库存一次减去 2 的示例。

```php
<?php

use App\Models\Goods;

$article = yield Goods::find(1);
yield $article->decrement('stock', 2);
```

### 同时更新其它字段的值

如果先使用 increment, decrement 后再指定其它属性的值后调用 save() 会造成两次查询，可以在递增递减的第三个参数 withAttributes 传递一个包含要更新的属性和值的数组。

以下是一个减少库存并把 status 字段改为 2 的示例：

```php
<?php

use App\Models\Goods;

// 这相当于 UPDATE `goods` SET `stock`=`stock`-1, `status`=2 WHERE `id`=1
$article = yield Goods::find(1);
yield $article->decrement('stock', 1, [
    'status' => 2
]);

//跳过第二个参数
yield $article->decrement('stock', withAttributes: [
    'status' => 2
]);
```

### 复杂的递增递减

调用 updateCounters() 方法可以实现同时为多个属性递增并递减。

第一个参数指定了要递增和递减的属性变更值，正值代表递增，负值代表递减。第二个可选参数指定同时更新的其它属性。

```php
yield $article->updateCounters([
    'stock' => -1,
    'sold' => 1
], [
    'status' => 2,
    'updated_at' => '2023-01-01 20:00:00'
])
```

## 增删改

模型提供了 insert, delete, update 三个方法用于插入，删除和修改。

### 新增

```php
$user = new User();
$user->username = 'little yang';
$user->nickname = '小杨';
yield $user->insert();

// 新增后 Model 自增主键字段会自动更新
echo "New user id is {$user->id}\n";
```

### 删除

```php
yield $user->delete();
```

### 修改

```php
$user = yield User::find(1);
$user->level = 'Gold';
$user->updated_at = '2023-01-01 20:00:00';
yield $user->update();
```

### save 方法

对于新增或修改的模型，可以直接调用 save() 方法，模型会自动判断当前是创建还是修改。

以下是一个获取或创建一个名为 Garlic 的标签的示例：

```php
$name = 'Garlic';

$tag = yield Tag::query()->where(['name'=>$name])->fetchOne();

if (!$tag) {
    $tag = new Tag();
    $tag->name = $name;
}

yield $tag->save();
```

## 虚拟属性

如果想要给模型增加一个并不存在的属性用于展示，或者该属性是经过已有属性计算而来，可以使用 get 开头的方法，方法名和字段使用驼峰命名法。

以下是一个获取 Image 模型实际图片访问地址的示例，url 是根据 path 计算而来：

```php
class Image extends Model {

    public function getDisplayUrl()
    {
        return 'https://www.example.com/images/'.$this->path;
    }

}
```

通过访问 displayUrl 属性来获得该虚拟属性的值。

```php
//https://www.example.com/images/helloworld.jpg
echo $image->displayUrl;
```

## 数组式访问

Model 实现了 ArrayAccess 和 IteratorAggregate 接口，可以像访问数组那样访问模型中的属性。

```php
$user = yield User::find(1);

echo "Show user {$user['username']}'s properties:\n";

foreach ($user as $key => $value) {
    echo "Property $key value is $value\n";
}
```
