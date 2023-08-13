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

默认情况下，模型会根据模型类来得出表名，如 App\Models\User 模型便会得出 `user` 表名，如果想自定义表名，在模型中定义属性 `$table` 指定表名即可。

```php
/**
 * User Model
 */
class User extends Model {

    protect $table = 'users';

}

```

## 配置所属数据库连接

如果想要指定模型所要使用的数据库连接，在模型中定义属性 `$connection`，指定 config/database.php 中配置的名称即可。

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

    protected $connection = 'data_store';

}

```

## 配置主键

模型会默认使用 `id` 作为模型的主键， 模型对单条数据的 find、保存、修改均会基于主键进行操作，当查询出一条模型时，确保其中包含主键的值，否则操作将失败，如果想要指定主键的字段名，在模型中定义属性 `$primaryKey`。

```php
/**
 * User Model
 */
class User extends Model {

    protected $primaryKey = 'uid';

}

```

## 数字值自增和自减

如果想要对模型的某个字段进行自增和自减的操作，比如给文章的浏览数 +1，应该使用模型提供的 increment, decrement 方法来递增和递减。

```php
<?php

use App\Models\Article;

// 这相当于 UPDATE `article` SET `views`=`views`+1 WHERE `id`=1
$article = Article::find(1);
$article->increment('views');
```

以下是错误的示例，因为在拿到 Article 实例后，views 可能已经被其它位置更新了，这样会导致本实例 views 的值 +1 后覆盖数据库已经被更新的值。

```php
<?php

use App\Models\Article;

// 错误示例，此方法存在并发问题
$article = Article::find(1);
$article->views += 1;
$article->update();
```

### 一次递增递减大于1的值
如果要递增或递减的数据不只是 1，可以传递第二个参数指定一个数字，以下是对商品库存一次减去 2 的示例。

```php
<?php

use App\Models\Goods;

$article = Goods::find(1);
$article->decrement('stock', 2);
```

### 同时更新其它字段的值

如果先使用 increment, decrement 后再指定其它属性的值后调用 save() 会造成两次查询，可以在递增递减的第三个参数 withAttributes 传递一个包含要更新的属性和值的数组。

以下是一个减少库存并把 status 字段改为 2 的示例：

```php
<?php

use App\Models\Goods;

// 这相当于 UPDATE `goods` SET `stock`=`stock`-1, `status`=2 WHERE `id`=1
$article = Goods::find(1);
$article->decrement('stock', 1, [
    'status' => 2
]);

// 在较新版本的 PHP 中可以使用新写法，跳过第二个参数
$article->decrement('stock', withAttributes: [
    'status' => 2
]);
```

### 复杂的递增递减

调用 updateCounters() 方法可以实现同时为多个属性递增并递减。

第一个参数指定了要递增和递减的属性变更值，正值代表递增，负值代表递减。第二个可选参数指定同时更新的其它属性。

```php
$article->updateCounters([
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
$user->insert();

// 新增后 Model 自增主键字段会自动更新
echo "New user id is {$user->id}\n";
```

### 删除

```php
$user->delete();
```

### 修改

```php
$user = User::find(1);
$user->level = 'Gold';
$user->updated_at = '2023-01-01 20:00:00';
$user->update();
```

### save 方法

对于新增或修改的模型，可以直接调用 save() 方法，模型会自动判断当前是创建还是修改。

以下是一个获取或创建一个名为 Garlic 的标签的示例：

```php
$name = 'Garlic';

$tag = Tag::query()->where(['name'=>$name])->fetchOne();

if (!$tag) {
    $tag = new Tag();
    $tag->name = $name;
}

$tag->save();
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
$user = User::find(1);

echo "Show user {$user['username']}'s properties:\n";

foreach ($user as $key => $value) {
    echo "Property $key value is $value\n";
}
```

## 模型事件

模型支持监听获取和处理数据的各个环节，以做到模型对应的数据在被查询、更新时处理的数据和业务联动，减轻实现负担和可能出现的遗漏。

### 支持的事件


| 事件名       | 常量                       | 触发时机                   |
|--------------|----------------------------|----------------------------|
| init         | Model::EVENT_INIT          | 模型实例化时               |
| retrieved    | Model::EVENT_RETRIEVED     | 模型查询到数据实例化后     |
| beforeCreate | Model::EVENT_BEFORE_CREATE | 模型实例插入数据库前       |
| created      | Model::EVENT_CREATED       | 模型实例插入数据库后       |
| beforeDelete | Model::EVENT_BEFORE_DELETE | 模型实例删除前             |
| deleted      | Model::EVENT_DELETED       | 模型实例数据从数据库删除后 |
| beforeUpdate | Model::EVENT_BEFORE_UPDATE | 模型实例更新前             |
| updated      | Model::EVENT_UPDATED       | 模型实例数据更新到数据库后 |

### 监听事件

有两种方式可以监听模型的事件。

#### 在模型中定义事件的同名方法

在模型中直接定义与事件同名的方法，在方法中通过 $this 便可以获得当前模型的实例，优点是方法简单，可以调用模型内 protected 和 private 的方法和属性。

示例：

```php
<?php

namespace App\Model;

use Wind\Db\Model;

class Post extends Model
{

    protected $table = 'posts';

    // 查询到数据事件
    protected function retrieved()
    {
        // date retrieved
    }

    // 模型更新前事件
    protected function beforeUpdate()
    {
        $this->updated_at = date('Y-m-d H:i:s');
    }

}

```

#### 通过静态方法将回调绑定到指定模型

通过模型的静态方法 `on` 和 `off` 可以绑定和解绑对应事件。

静态方法绑定的优点是比较灵活，可以绑定也可以解绑，当然缺点也很明显，绑定与解绑只在所在进程中生效，如果在某个进程的业务中进行绑定，则其它进程的执行不会触发这个绑定的回调，容易产生差异问题。为了防止这个问题，一般只在框架初始化、或者模型的启动事件中进行绑定。

另一个问题是静态方法绑定属于外部调用，在回调中无法调用模型内 protected 和 private 的方法和属性。

模型的 `static boot()` 方法就是模型的启动事件方法，以下是一个在模型的启动事件中进行静态方法绑定的示例，模型的启动方法在进程中只在初次调用模型时被执行，在这里进行事件绑定是很稳妥的。

```php
//在 User 模型插入前自动设置注册时间
<?php

namespace App\Model;

use Wind\Db\Behaviors\DatetimeBehavior;
use Wind\Db\Model;
use Wind\Db\Modifier\DatetimeModifier;

/**
 * User Model
 * @property int $id
 * @property string $username
 * @property string $register_at
 * ..
 */
class User extends Model
{

    protected $table = 'users';

    // 模型的启动事件
    protected static function boot()
    {
        self::on(Model::EVENT_BEFORE_CREATE, function(User $user) {
            $user->register_at = date('Y-m-d H:i:s');
        });
    }

}

```

## 模型扩展

通过模型扩展，可以直接为模型引入新功能，比如预设好的事件处理等等。

Wind 框架目前内设了两个模型扩展，分别是 `\Wind\Db\Modifier\TimestampModifier` 和 `\Wind\Db\Modifier\DatetimeModifier`，它们的作用是在模型对应事件中预设时间字段的值。

以 `DatetimeModifier` 为例，以下是在模型插入和更新时，预设创建和更新时间字段值的示例：

```php
<?php

namespace App\Model;

use Wind\Db\Model;
use Wind\Db\Modifier\DatetimeModifier;

class Post extends Model
{

    use DatetimeModifier;

    protected $table = 'posts';

    protected $datetimeAttributes = [
        Model::EVENT_BEFORE_CREATE => ['created_at', 'updated_at'],
        Model::EVENT_BEFORE_UPDATE => ['updated_at']
    ];

    //protected $datetimeFormat = 'Y-m-d H:i:s';

}

```

模型的扩展是一个 `trait`，通过 `use` 引入模型类中后，再加上需要的配置即可（有的扩展可能不需要配置）。

例中的 `DatetimeModifier` 被定义为一个日期时间修改器，它的作用是在模型对应事件触发时，使用当前时间填充指定的字段。它支持两个模型配置属性，分别是 `$datetimeAttributes` 用来指定模型对应事件触发时应该填充哪些字段，以及 `$datetimeFormat` 用来指定时间的格式，`$datetimeFormat` 的默认值为 `'Y-m-d H:i:s'`。

`TimestampModifier` 与 `DatetimeModifier` 类似，不同的是它使用 `time()` 生成的时间戳填充字段，仅支持 `$timestampAttributes` 配置字段。

### 实现一个模型扩展

Todo
