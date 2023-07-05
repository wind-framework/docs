# 配置 <!-- {docsify-ignore} -->

配置是应用程序框架的核心部分，Wind 框架通过 Config 类和 .env 文件来处理配置。

## 配置文件

配置文件统一存放于 config 目录，config 目录处于 start.php（启动文件）中定义的 BASE_DIR 中。

每个配置文件为一个 php 文件，内容如下：
```php
<?php
return [
    'key1' => 'Config Value',
    'key2' => [
        'keyx' => 'X Value',
        'keyy' => 'Y Value'
    ]
];
```

## 读取配置

如配置文件名称为 orange.php，则通过 Config 类的 get() 方法传入 `配置文件名无后缀.键名1.键名2`，多级用符号 `.` 连接，以此类推即可。

如要获取 keyx 的值：
```php
<?php
//通过 di() 的全局容器单例获取 Config 实例
$config = di()->get(\Wind\Base\Config::class);
$x = $config->get('orange.key2.keyx');
```

可以传入第二个参数以获取没有相应配置项时的默认值。
```php
$config->get('orange.key3', 'Default Value');
```

没有传递默认值且没有该配置项时，将返回 `null`。

默认值不能针对一级的文件名，如 `$config->get('some');` 若没有 some.php 配置文件将抛出异常。

## 提前加载配置

可使用 `load()` 方法提交加载配置。

```php
$config->load('thing');
```

此时配置类将提前加载 thing.php 的配置内容，并返回整个配置文件的值。

## 判断配置文件是否存在

可通过 `exists()` 方法判断配置文件是否存在。

```php
<?php
if ($config->exists('foo')) {
    echo 'Config foo exists.';
} else {
    echo 'Config foo not exists.';
}
```
## 环境变量

应用程序在不同环境下的差异配置，如数据库、缓存地址。AppKey 或密钥等，一般不建议直接存放于 config 目录下的 php 配置文件中。

我们可以在程序根目录（start.php 同级）下放置一个 .env 文件，里面有存放这些环境配置，然后在 php 配置文件中通过 env() 函数获取配置，
自己的程序中建议依然使用 Config 类的方法读取配置。

**.env 文件**
```dotenv
DB_HOST=127.0.0.1
DB_PASSWORD="#*9283912839123"
```

**config/foo.php**
```php
<?php
return [
    'host' => env('DB_HOST'),
    'password' => env('DB_PASSWORD'),
    'port' => env('DB_PORT', 3306)
];
```

env() 函数的第二个参数同样支持默认值返回。

在布署时将不同环境的配置文件复制为根目录的 .env 文件即可。