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

Todo

#### AND, OR 嵌套
#### IN, NOT INT
#### BETWEEN
#### 其它查询条件
#### 使用原始语句替代条件

```
Simple condition:
['id'=>100]
means:
`id`=100

First element is 'AND', 'OR' mean condition connect method:
['name'=>'hello', 'nick'=>'world'] -> `name`='hello' AND `nick`='world'
['OR', 'name'=>'hello', 'nick'=>'world'] -> `name`='hello' OR `nick`='world'

AND, OR support multiple nested:
['name'=>'hello', ['OR', 'c'=>1, 'd'=>2]] -> `name`='hello' AND (`c`=1 OR `d`=2)

IN, NOT IN:
['name'=>['a', 'b', 'c']] -> `name` IN('a', 'b', 'c') AND
['name !'=>['a', 'b']] -> `name` NOT IN('a', 'b')

BETWEEN:
['id BETWEEN'=>[100, 999]] -> `id` BETWEEN 100 AND 999

Other symbols:
=, !=, >, >=, <, <=, EXISTS, NOT EXISTS and others
['id >='=>100, 'live EXISTS'=>'system'] -> `id`>=100 AND `live` EXISTS ('system')
```

### JOIN 查询

Todo

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
