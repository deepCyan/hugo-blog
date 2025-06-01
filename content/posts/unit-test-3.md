+++
date = '2023-04-05T08:06:08+08:00'
draft = false
title = '单元测试(三)'
summary = '单测第三部分, 这一部分主要介绍一下 Hyperf 和 Laravel 的微小差异'
+++

### 配置文件

首先看一下 phpunit.xml 的问题

在 2.2 的版本，会出现配置 phpunit.xml 不生效的问题，最终解决的方案是在 test/bootstrap.php 中手动设置 env 配置，大概这样

```php
<?php

declare(strict_types=1);

ini_set('display_errors', 'on');
ini_set('display_startup_errors', 'on');

error_reporting(E_ALL);
date_default_timezone_set('Asia/Shanghai');

! defined('BASE_PATH') && define('BASE_PATH', dirname(__DIR__, 1));
! defined('SWOOLE_HOOK_FLAGS') && define('SWOOLE_HOOK_FLAGS', SWOOLE_HOOK_ALL);

Swoole\Runtime::enableCoroutine(true);

require BASE_PATH . '/vendor/autoload.php';

Hyperf\Di\ClassLoader::init();

/** @var \Hyperf\Di\Container $container */
$container = require BASE_PATH . '/config/container.php';

// 这样将 env 覆盖进去即可
putenv('APP_ENV=testing');
putenv('DB_DRIVER=sqlite');
putenv('DB_DATABASE=:memory:');
putenv('REDIS_PORT=16380');
putenv('REDIS_PORT_ROOM=16380');
putenv('REDIS_PORT_USER=16380');
$container->get(Hyperf\Contract\ApplicationInterface::class);
```

### DB Driver
然后就是要处理一下 Mock 数据库的问题，由于官方并没有提供 Sqlite 的 Driver，所以需要使用私人包，这里感谢一下 N 老师
`composer require fangx/sqlite-driver --dev`

### 基类
由于 Hyperf 并没有提供类似 Laravel 的 RefreshDatabase trait，所以这一部分需要自己实现，包括自动执行 migration

可以构建如下的基类

```php
<?php

declare(strict_types=1);
/**
 * This file is part of Hyperf.
 *
 * @link     http://gitlab.iusns.com/fox/funbox-server
 * @document https://hyperf.wiki
 * @contact  group@hyperf.io
 * @license  https://github.com/hyperf/hyperf/blob/master/LICENSE
 */
namespace HyperfTest;

/**
 * @internal
 * @coversNothing
 */
abstract class BaseTest extends TestCase
{
    use MakeServiceTrait;

    public function setUp(): void
    {
        $this->command('migrate', [
            '--path' => $this->getMigrationsPath(),
        ]);
        $mockLoggerFactory = \Mockery::mock(LoggerFactory::class, function (MockInterface $mock) {
            $nullLogger = ApplicationContext::getContainer()->get(NullLogger::class);
            $mock->shouldReceive('get')->withAnyArgs()->andReturn($nullLogger);
        });
        /** @var \Hyperf\Di\Container $container */
        $container = ApplicationContext::getContainer();
        $container->set(LoggerFactory::class, $mockLoggerFactory);
        ApplicationContext::setContainer($container);
        parent::setUp();
    }

    protected function tearDown(): void
    {
        ApplicationContext::getContainer()->get(RedisFactory::class)->get('room')->flushAll();
        $this->command('migrate:refresh', [
            '--path' => $this->getMigrationsPath(),
        ]);
    }

    protected function command($command, $parameters = [])
    {
        $app = ApplicationContext::getContainer()
            ->get(ApplicationInterface::class);

        $app->setAutoExit(false);

        return $app->find($command)->run(make(ArrayInput::class, [$parameters]), make(NullOutput::class));
    }

    private function getMigrationsPath()
    {
        return 'migrations/unit_test';
    }
}
```