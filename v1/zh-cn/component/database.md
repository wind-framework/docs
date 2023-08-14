# 数据库

Wind框架的数据库组件具有以下特点：

1. 异步查询，隐式协程、同步编程方式支持。
2. 连接池、连接复用，高并发支持。
3. 无原生PDO、mysqli扩展的依赖。

## 安装

```
composer require wind-framework/db:1.0.x-dev
```

## 配置

数据库的配置文件在 config/database.php，如果没有该文件请先创建该文件。

配置示例：
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
    ]
];

```

配置详解：

配置为一个多维数组，数组的索引代表配置组的名称，组内配置如下表：

| 配置项 | 类型 | 含义 | 默认值 |
|---|---|---|---|
| type | string | 数据库类型，目前仅支持 MySQL | mysql |
| host | string | 数据库服务器地址 |  |
| port | int | 数据库服务端口 | 3306 |
| database | string | 选择的数据库 |  |
| username | string | 数据库用户名 |  |
| password | string | 数据库密码 |  |
| prefix | string | 数据表前缀 |  |
| charset | string | 数据库编码集 | utf8mb4 |
| collation | string | 数据库编码 | utf8mb4_general_ci |
| pool | array | 连接池配置 |  |
| pool.max_connections | int | 连接池允许的最大连接数 | 不限制 |
| pool.max_idle_time | int | 池中的连接限制多久后被关闭（秒） | 30 |

数据库默认会使用 `default` 组的配置进行连接。

## 查询

使用原始的 SQL 进行查询时，需使用 `Wind\Db\Db` 类。

### 执行SQL语句

`query()` 方法返回一个 `MysqlResult` 对象，因为 `MysqlResult` 对象实现了 `Traversable` 的特性，所以可以使用 foreach 遍历其中的每一行。

```php
use Wind\Db\Db;

$result = Db::query('SELECT * FROM articles LIMIT 1');
foreach ($result as $row) {
    print_r($row);
}
```

### 查询单条数据

```php
$row = Db::fetchOne('SELECT * FROM articles LIMIT 1');
```

### 查询多条数据

```php
$rows = Db::fetchAll('SELECT * FROM articles LIMIT 10');
```

### 执行查询并获取受影响的行数

`execute()` 适用于更新、删除数据等不需要获取表中数据的操作。
`execute()` 方法同样返回 `MysqlResult` 对象，可通过 `MysqlResult` 的 `getRowsCount()` 方法获得受影响的条数。

```php
$result = Db::execute('UPDATE articles SET `status`=2 WHERE views<1');
$result->getRowCount();
```

## 查询构造器

使用查询构造器来构建查询，代码将更加直观，输入的值也会经过构造器的处理，更加安全。

使用查询构造器只需通过 `table()` 方法选定一张表并返回一个 `QueryBuilder` 对象，即可使用 `QueryBuilder` 的链式方法进行查询。

### 查询单条数据

```php
// SELECT * FROM `articles` WHERE `id`=1
$row = Db::table('articles')->where(['id'=>1])->fetchOne();
```

### 查询某个字段的值

使用 `scalar()` 来指定要取出具体某一个字段的值，而不是返回整行，`scalar()` 默认会返回查询结果中第一个字段的值。

使用 scalar() 方法时，最好使用 select() 方法限定查询的字段，不要让不相关的字段来浪费性能。

```php
// SELECT `name` FROM `users` WHERE `id`=1
$name = Db::table('users')
    ->select('name')
    ->where(['id'=>1])
    ->scalar('name');
```

### 查询多条数据

使用 `fetchAll()` 查询多条数据，返回结果为一个包含匹配结果的列表数组。

```php
// SELECT `id`, `title`, `views` FROM `articles` WHERE `status`=1 AND `id`=1
$rows = Db::table('articles')
    ->select('id, title, views')
    ->where(['status'=>1, 'views >'=>1])
    ->fetchAll();
```

### 以某个字段的值作为索性

通过 `indexBy()` 指定取某个字段作为多行结果的索引，相同的索引将只返回最后一个匹配的行。

```php
$rowsIndexByUserId = Db::table('articles')
    ->where(['views >'=>1])
    ->indexBy('user_id')
    ->fetchAll();
```

### 查询某一列

`fetchColumn()` 可以只将某一列的结果作为数组返回，以下是一个获取所有浏览数大于1的文章ID列表的示例。

> 注意查询某一列时最好使用 `select()` 指定涉及到的字段，避免不需要的字段数据带来的性能浪费。
> `fetchColumn()` 的 `$col` 参数可以是字段名（字符串），也可以指定为第几个字段（整型）。

```php
$articleIds = Db::table('articles')
    ->select('id')
    ->where(['views >'=>1])
    ->fetchColumn('id');
```

也可以结合 `indexBy()` 指定一个字段作为键，另一个字段作为值的键值对数组。

以下是一个获取用户id 作为索引，nickname 作为值的示例：

```php
$userNickMap = Db::table('users')
    ->select('user_id, nickname')
    ->indexBy('user_id')
    ->fetchColumn('nickname');
```

### 插入数据

使用 `insert()` 插入数据并获取新生成的自增ID。

```php
$insertId = Db::table('users')->insert([
    'nickname' => 'Pader',
    'email' =>'mail@example.com'
]);
```

### 更新数据

使用 `update()` 更新数据，更新数据时可以使用 `^` 符号作为字段前缀来标记值是一个原始的语句，这样查询构造器就不会对值进行 SQL 转义处理。

`update()` 方法返回受影响的行数。

```php
$affectedRowCount = Db::table('articles')
    ->where(['id'=>1])
    ->update(['updated_at'=>time(), '^views'=>'views+1']);
```

> 也可以使用 \Wind\Db\Expression 类作为值来表示一个原始SQL。

### 删除数据

`delete()` 方法删除数据并返回受影响的行数。

```php
$affectedRowCount = Db::table('articles')->where(['id'=>1])->delete();
```

### 查询条件

通过 QueryBuilder 的 `where()` 和 `having()` 方法可以指定查询条件，查询条件是一个多维数组。

一个简单的键值对数组就可以组成 where 查询条件。

```php
// SELECT * FROM `users` WHERE `id`=1
DB::table('users')->where(['id'=>1])->fetchOne();
```

多个键值对将以 AND 连接在一起，以下将用 `$condition` 代表 `where()` 和 `having()` 中的条件。

```php
// `type`=1 AND `status`=2 AND `name`='John'
$condition = ['type'=>1, 'status'=>2, 'name'=>'John'];
```

使用 OR 作为数组的第一个元素让条件以 OR 的形式连接在一起。

```php
// `type`=1 AND `status`=2 AND `name`='John'
$condition = ['type'=>1, 'status'=>2, 'name'=>'John'];
```

#### AND, OR 嵌套

有时候查询条件并不是简单的 AND 或者 OR，如果想组件 AND 和 OR，可以在下级数组的开头指定嵌套方式。

```php
// `type`=1 AND (`status`=2 OR `visible`='display')
$condition = ['type'=>1, ['OR', 'status'=>2, 'visible'=>'display']];
```

或者

```php
// `type`=1 OR (`status`=2 AND `visible`='display')
$condition = ['OR', 'type'=>1, ['status'=>2, 'visible'=>'display']];
```

也可以进行很多层的嵌套：

```php
$condition = [
    'type' => 1,
    [
        'OR',
        'status' => 2,
        'visible' => 'display',
        [
            'views >' => 10,
            'user_id' => 3
        ]
    ]
];
```

最终语句（格式化后）：

```sql
`type`=1
AND (
    `status`=2
    OR `visible`='display'
    OR (
        `views`>10
        AND `user_id`=3
    )
)
```

#### IN, NOT IN

当条件字段指向的值是数组时，使会转化为 IN 查询。

```php
// `id` IN(1, 2, 3)
$condition = [
    'id' => [1, 2, 3]
];
```

在字段的后方跟上 ` !`（注意符号前有一个空格），条件就会变为 NOT IN。

```php
// `id` NOT IN(1, 2, 3)
$condition = [
    'id !' => [1, 2, 3]
];
```

#### BETWEEN

```php
// WHERE `id` BETWEEN 1 AND 99
$condition = [
    'id between' => [1, 99]
];
```

#### 其它查询条件

在条件数组的键名中加上空格和对应符号，就可以生成对应的查询，如：

```php
// `id`!=1
$condition = ['id !=' => 1];

// `id`<>1
$condition = ['id <>' => 1];

// `id`>1
$condition = ['id >' => 1];

// `id`>=1
$condition = ['id >=' => 1];

// `id`<=1
$condition = ['id <=' => 1];

// `id` LIKE 'peace%'
$condition = ['id like' => 'peace%'];

// `id` NOT LIKE 'peace%'
$condition = ['id not like' => 'peace%'];
```

#### 使用原始语句替代条件

如果在使用查询构造器时，想要直接使用原始的 SQL 语句来构建查询，有以下三种方法：

- 不指定键名
- 使用 `^` 符号作为字段名的前缀
- 使用 `\Wind\Db\Expression` 对象作为值

不指定键名时，数组的值将视为原始的 SQL 语句，以下示例的第二个元素未指定数组键名。

```php
// `type`=1 AND updated_at IS NOT NULL
$condition = ['type'=>1, 'updated_at IS NOT NULL'];
```

使用 `^` 前缀时，代表字段的值是一个原始语句。

```php
// `updated_at` IS NOT NULL
$condition = ['^updated_at' => 'IS NOT NULL'];
```

使用 `\Wind\Db\Expression` 对象作为输入也同样不会被转义。

```php
// `updated_at` IS NOT NULL
$condition = ['updated_at' => new Expression('IS NOT NULL')];
```

### JOIN 查询

QueryBuilder 支持简单的 JOIN 查询。

在使用 join 查询时建议使用 `alias()` 方法先为当前表取一个别名，同时在 join() 参数中指定连接表的别名。

```php
// SELECT * FROM `users` `u` LEFT JOIN `user_profiles` p ON `p`.`id`=`u`.`id` WHERE `u`.`id`=1
Db::table('users')
    ->alias('u')
    ->leftJoin('user_profiles', 'p.id=u.id', 'p')
    ->where(['u.id'=>1])
    ->fetchOne();
```

除了 `leftJoin()` 外，还支持 `rightJoin()`，`innerJoin`， `outerJoin()` 参数相同。


### UNION 查询

使用 `union()` 方法可以直接链式创建出一条 UNION 的查询。

```php
// SELECT * FROM `users` WHERE `type`=1 UNION SELECT * FROM `users` WHERE `type`=2
Db::table('users')
    ->where(['type'=>1])
    ->union()
    ->from('users')
    ->where(['type'=>2])
    ->fetchAll();
```

指定 `union($all)` 的 `$all` 参数为 `true`，将生成 `UNION ALL` 查询。

```php
// SELECT * FROM `users` WHERE `type`=1 UNION ALL SELECT * FROM `users` WHERE `type`=2
Db::table('users')
    ->where(['type'=>1])
    ->union(true)
    ->from('users')
    ->where(['type'=>2])
    ->fetchAll();
```

## 查询其它数据库的连接

通过 `Db::connection()` 方法可获取指定配置组的数据库连接实例，通过实例的方法进行操作。

之后的方法与 `\Wind\Db\Db` 中的静态方法一致，实际上 Db 中的静态方法就是 `Db::connection()` 获取默认连接后的封装。

```php
$article1 = Db::connection('blog2')
    ->fetchOne('SELECT * FROM articles WHERE id=1');

$article2 = Db::connection('blog2')
    ->table('articles')
    ->where(['id'=>2])
    ->fetchOne();
```
