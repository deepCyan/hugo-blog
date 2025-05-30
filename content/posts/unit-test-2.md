+++
date = '2023-04-03T08:50:23+08:00'
draft = false
title = '单元测试(二)'
summary = '单测第二部分, 这一部分主要介绍一下 Laravel 如何进行单元测试'
+++

### 额外依赖

大部分按照文档配置即可，只有一些细节需要我们注意一下

首先是数据库和 Redis 的 Mock 问题，但是 Mock 这两个东西是非常难的事情

Redis 有数组实现的 mock 包，但是对 phpredis 也有兼容问题，所以建议 Redis 使用外部依赖，这也是运行测试唯一的外部依赖实例

我的做法是启动一个 redis docker 并在 phpunit.xml 中覆盖对应配置

至于数据库就更加困难了，但是我们可以使用基于内存的 sqlite 来替换 MySQL ，这就需要我们安装了 PDO SQLITE 的扩展，由于是基于内存的，所以速度也非常快

### 数据库迁移

但是需要注意的是 sqlite 不支持修改表的操作，所以建议为单元测试单独划分 migration 并在测试环境中加载

例如新建了一张 users 表，那么实际需要两个迁移文件，分别在 database/migrations 和 database/migrations/test 各自新建，在对 users 表进行修改的时候，则需要新建一个修改 users 的 migration，并且在 database/migrations/test 中对原有的 users 进行修改，也就是在 database/migrations/test 中每张表只有一个相关的 migration

设置完成后我们要让框架在测试环境加载对应的迁移文件，而不是默认的

在 AppServiceProvider 的 boot 方法增加如下代码

```php
if (config('app.env') === 'testing') {
    $this->loadMigrationsFrom(__DIR__ . '/../../database/migrations/test');
}
```

### 设置 phpunit.xml
在设置数据库迁移的时候，注意到 app.env 是可以为 testing 的，这也是判断当前是否为测试环境的方式，这个环境变量实际上是通过 phpunit.xml 设置的

在运行单元测试的时候，phpunit.xml 中的 env 配置可以覆盖 .env 中的数据

我的配置文件如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" backupGlobals="false" backupStaticAttributes="false" bootstrap="vendor/autoload.php" colors="true" convertErrorsToExceptions="true" convertNoticesToExceptions="true" convertWarningsToExceptions="true" processIsolation="false" stopOnFailure="false" xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd">
  <coverage processUncoveredFiles="true">
    <include>
      <directory suffix=".php">./app</directory>
    </include>
  </coverage>
  <testsuites>
    <testsuite name="Unit">
      <directory suffix="Test.php">./tests/</directory>
      <exclude>./tests/Mysql/</exclude>
    </testsuite>
  </testsuites>
  <php>
    <env name="APP_ENV" value="testing" force="true"/>
    <env name="DB_CONNECTION" value="sqlite_testing" force="true"/>
    <env name="BCRYPT_ROUNDS" value="4" force="true"/>
    <env name="CACHE_DRIVER" value="array" force="true"/>
    <env name="SESSION_DRIVER" value="array" force="true"/>
    <env name="QUEUE_DRIVER" value="sync" force="true"/>
    <env name="MAIL_DRIVER" value="array" force="true"/>
    <env name="REDIS_CLIENT" value="mock" force="true"/>
    <env name="TELESCOPE_ENABLED" value="false" force="true"/>
    <env name="LOG_CHANNEL" value="syslog" force="true"/>
  </php>
</phpunit>
```

需要注意的是这几点
- APP_ENV 设置为 testing
- DB_CONNECTION 的值为 sqlite_testing，这需要在 database 的配置文件中增加一个配置如下
  ```php
  'connections' => [
        'sqlite_testing' => [
            'driver' => 'sqlite',
            'database' => ':memory:', // :memory: 会指明使用内存型的数据库
            'prefix' => '',
        ],
    ]
  ```

### 写一个简单的测试

万事具备，让我们写一个简单的 case

为了演示，我准备了如下代码

```php
<?php

namespace App\Services;

use App\Exceptions\ApiException;
use App\Models\User;

class DemoService
{
    const CHECK_USER_NAME_URL = 'this_is_fake_url';
    
    protected $httpService;

    public function __construct(HttpService $httpService)
    {
        $this->httpService = $httpService;
    }

    public function checkUserName(string $userId)
    {
        /** @var User|null $user */
        $user = User::query()->select('id', 'name')->first();
        if (empty($user)) {
            throw new ApiException('用户不存在');
        }
        $res = $this->httpService->get(self::CHECK_USER_NAME_URL, ['name' => $user->name]);
        if (! is_array($res)) {
            throw new ApiException('验证失败');
        }
        if (isset($res['code']) && $res['code'] == 0) {
            return true;
        }
        return false;
    }
}
```

代码很简单，但是内容比较全面，我们需要数据表，网络请求，大部分测试这就足够了，然后看为这个方法编写的测试用例

```php
<?php

namespace Tests\Unit;

use App\Format\UserFormat;
use App\Services\DemoService;
use App\Services\HttpService;
use App\Services\UserService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Mockery\MockInterface;
use Tests\TestCase;

class DemoTest extends TestCase
{
    use RefreshDatabase;

    public function testCheckUserName()
    {
        $phone = '18131699918';
        $password = '1234567';
        $userService = new UserService();
        $userFormat = new UserFormat([
            'phone' => $phone,
            'password' => $password,
            'password_question_type' => 1,
            'password_question_answer' => '111',
        ]);
        $reg = $userService->reg($userFormat, true);
        $this->assertEquals($reg['user_id'], 1);
        $httpService = \Mockery::mock(HttpService::class, function (MockInterface $mock) {
            $mock->shouldReceive('get')->andReturn(['code' => 0]);
        });
        $demoService = new DemoService($httpService);
        $check = $demoService->checkUserName($reg['user_id']);
        $this->assertTrue($check);
    }
}
```

我们来逐一解析这段测试代码的含义
- 首先测试类，需要继承 Tests\TestCase，以使用 phpunit
- 其次引用了一个 RefreshDatabase trait，这个 trait 可以帮助我们在运行每一个 case 后清空数据库，这对于构建测试是非常重要的
- 定义一个测试方法，注意方法名以 test 开头
- 然后我们注册了一个用户，其实这是准备数据的阶段，实际上这也测试了 userService 的 reg 方法
- $this->assertEquals() 是 phpunit 提供的断言方法，稍后的运行结果会展示出它的意义
- 下面 mock 了 HttpService，观察 DemoService，会发现 CHECK_USER_NAME_URL 是不可访问的，但是用 Mock 可以很轻松的解决这个问题，设置被 mock 的 HttpService 的 get 方法会返回 ['code' => 0], 其实 Mockery 还有很丰富的方法，可以深入探索
- 从这里也能看到使用依赖注入的好处，如果在 checkUserName 方法内部去 new HttpService，那这个代码就非常难以测试
- 最后一个 $this->assertTrue($check) 的断言，预期在当前条件下，checkUserName 必然返回 true

看一下运行测试的结果
![Laravel Test Result](/images/laravel-unit-test-result.png)
可以发现测试通过，并且下方显示有 1 test 表示一个测试用例，3 assertions 表示有三个断言，实际上 shouldReceive 也表示了一个断言，那么在以后的代码更新中，只要运行 composer test 就可以运行所有的测试，修改代码所疏忽的问题都会一一暴露（当然前提是有完善的测试用例）