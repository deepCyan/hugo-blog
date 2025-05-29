+++
date = '2023-01-31T19:46:16+08:00'
draft = false
title = 'Jager in Laravel'
summary = '论如何将 Jaeger 放进 Laravel'
+++

### 简介

Jaeger 是一款优秀的链路追踪工具（Tracing），一般来说，它真正发挥能力，是在多服务的串联的场景（如微服务），但即便是简单的单体应用，我们也可以借助它来进行服务监控和 debug，以典型的 web 请求为例，借助它，我们可以观察到

- 请求参数和响应

- 请求中涉及到的 DB / Redis / Http 等 IO 请求的时序和耗时

- 两个请求（Trace）的比对

- 简单的聚合

本文将介绍如何在 Laravel（8.0）中接入 Jaeger


### 三个简单概念

以一个单体应用的 HTTP 请求为例

Trace: 一个请求就是一个 trace

Span: 一个请求中的一个片段（DB / Redis..）可以构成一个 Span, 多个 Span 就会组成一个 Trace

Tag: 描述 Span ，如描述一个 DB Query Span 的 tag 可以有 sql 语句，执行时间等等

### 接入

Jaeger 提供了常见语言的客户端，但是需要注意的是有两种协议， opentracing 和 opentelemetry ，一般情况下两种协议不能互通（但是江湖传言有个 bridge）

首先部署服务端，使用官方提供的 all-in-one 是最方便的，官方地址 https://www.jaegertracing.io/docs/1.41/getting-started/#all-in-one

启动容器后，访问 16686 端口即可

注意到容器支持 UDP/TCP 两种通信协议，在实践中发现 UDP 协议会有包大小限制，如果链路过长导致上传失败可以选择 TCP 端口

然后进行客户端接入

需要指出的是，由于 Jaeger 需要监控每一次 IO 操作，那么天然就会对代码有侵入性，所以并不是任何工程都可以轻松的接入的，但是得益于 Laravel 优秀的设计，我们可以借助事件（Event）系统和依赖注入（DI）来实现

- 首先引入 PHP 的 jaeger 客户端 composer require jonahgeorge/jaeger-client-php

- 新建 App\Http\Middleware\JargerMiddleware, handle 方法如下

    ```php
    public function handle($request, Closure $next)
    {
        $config = new Config(
            [
                'sampler' => [
                    'type' => Jaeger\SAMPLER_TYPE_CONST,
                    'param' => 1,
                ],
                'logging' => true,
                'local_agent' => [
                    'reporting_host' => env('JARGER_REPORTING_HOST', 'localhost'),
                    'reporting_port' => env('JARGER_REPORTING_PORT', 6832),
                ],
                'dispatch_mode' => Config::JAEGER_OVER_BINARY_UDP,
            ],
            'jingyao-' . config('app.env')
        );
        $config->initializeTracer();
        $tracer = GlobalTracer::get();
        $scope = $tracer->startActiveSpan('webHttp', []);
        $span = $scope->getSpan();
        $span->setTag('type', 'http');
        $span->setTag('request_method', $request->method());
        $span->setTag('request_path', $request->path());
        $span->setTag('request_uri', $request->getRequestUri());
        $span->setTag('request_ip', $request->ip());
        $response = $next($request);
        $scope->close();
        $tracer->flush();
        return $response;
    }
    ```
    将此中间件设置为全局生效，Trace 就从这里开始

- 在 App\Providers\AppServiceProvider 中的 boot 方法添加代码，其实在哪个 Provider 都无所谓，只要在 boot 方法即可，此时核心类已经加载完成
  ```php
  DB::listen(function ($query) {
    $tracer = GlobalTracer::get();
    $activeSpan = $tracer->getActiveSpan();
    $endTime = microtime(true);
    $span = $tracer->startSpan('db.query', [
        'start_time' => (int)(($endTime - $query->time / 1000) * 1000 * 1000),
        'child_of' => $activeSpan->getContext(),
    ]);
    $sql = $query->sql;
    if (! Arr::isAssoc($query->bindings)) {
        foreach ($query->bindings as $key => $value) {
            $sql = Str::replaceFirst('?', "'{$value}'", $sql);
        }
    }
    $span->setTag('db.sql', $sql);
    $span->setTag('db.query_time', $query->time);
    $span->finish();
    });
  ```
  这部分代码的功能是记录 sql 内容

- 其实到这里就已经是一个相对完善的部分了，在简单的业务中已经足够使用了，但是我们可以继续深入，为 Redis 也添加监控，由于没有 Redis 的事件，所以我们自己来替换 Redis，当然你可以自己封装一个 Redis，但是如果是一个中途接入的工程，这显然不现实，那么我们的替换步骤如下
- 新建一个 App\Replace 目录，其实目录名字随意啦
- 新建 App\Replaces\Redis\ReplaceRedisManager , 继承 Illuminate\Redis\RedisManager，内容如下
  ```php
    namespace App\Replaces\Redis;

    use Illuminate\Redis\Connectors\PredisConnector;
    use Illuminate\Redis\RedisManager;

    class ReplaceRedisManager extends RedisManager
    {
        /**
        * Get the connector instance for the current driver.
        *
        * @return \Illuminate\Contracts\Redis\Connector
        */
        protected function connector()
        {
            $customCreator = $this->customCreators[$this->driver] ?? null;

            if ($customCreator) {
                return $customCreator();
            }

            switch ($this->driver) {
                case 'predis':
                    return new PredisConnector();
                case 'phpredis':
                    return new ReplacePhpRedisConnector();
            }
            throw new \Exception('Not found redis driver ' . $this->driver);
        }
    }
  ```
- 新建 App\Replaces\Redis\ReplacePhpRedisConnector , 继承 Illuminate\Redis\Connectors\PhpRedisConnector，内容如下
  ```php
    namespace App\Replaces\Redis;

    use Illuminate\Redis\Connectors\PhpRedisConnector;
    use Illuminate\Support\Arr;

    class ReplacePhpRedisConnector extends PhpRedisConnector
    {
        /**
        * Create a new clustered PhpRedis connection.
        *
        * @return \Illuminate\Redis\Connections\PhpRedisConnection
        */
        public function connect(array $config, array $options)
        {
            $connector = function () use ($config, $options) {
                return $this->createClient(array_merge(
                    $config,
                    $options,
                    Arr::pull($config, 'options', [])
                ));
            };

            return new ReplacePhpRedisConnection($connector(), $connector, $config);
        }
    }
  ```
- 新建 App\Replaces\Redis\ReplacePhpRedisConnection 继承 Illuminate\Redis\Connections\PhpRedisConnection，内容如下
  ```php
    class ReplacePhpRedisConnection extends PhpRedisConnection
    {
        public function __construct($client, callable $connector = null, array $config = [])
        {
            parent::__construct($client, $connector, $config);
        }

        public function get($key)
        {
            return $this->track(function () use ($key) {
                return parent::get($key);
            }, __FUNCTION__, $key);
        }

        public function set($key, $value, $expireResolution = null, $expireTTL = null, $flag = null)
        {
            return $this->track(function () use ($key, $value, $expireResolution, $expireTTL, $flag) {
                return parent::set($key, $value, $expireResolution, $expireTTL, $flag);
            }, __FUNCTION__, $key);
        }

        private function track(callable $callback, string $type, string $key)
        {
            if (config('app.env') !== 'test') {
                return $callback();
            }
            $tracer = GlobalTracer::get();
            $activeSpan = $tracer->getActiveSpan();
            $startTime = microtime(true);
            $res = $callback();
            $endTime = microtime(true);
            $span = $tracer->startSpan('redis.' . $type, [
                'child_of' => $activeSpan->getContext(),
                'start_time' => (int)($startTime * 1000 * 1000),
            ]);
            $span->setTag('redis.key', $key);
            $span->setTag('redis.client_time', $endTime - $startTime);
            $span->finish();
            return $res;
        }
    }
  ```
  这里只重写了 get 和 set 方法，可以将其他方法重写进来

- 新建 App\Providers\ReplaceRedisServiceProvider ，内容如下
  ```php
    namespace App\Providers;

    use App\Replaces\Redis\ReplaceRedisManager;
    use Illuminate\Contracts\Support\DeferrableProvider;
    use Illuminate\Support\Arr;
    use Illuminate\Support\ServiceProvider;

    class ReplaceRedisServiceProvider extends ServiceProvider implements DeferrableProvider
    {
        /**
        * Register the service provider.
        */
        public function register()
        {
            $this->app->singleton('redis', function ($app) {
                $config = $app->make('config')->get('database.redis', []);

                return new ReplaceRedisManager($app, Arr::pull($config, 'client', 'phpredis'), $config);
            });

            $this->app->bind('redis.connection', function ($app) {
                return $app['redis']->connection();
            });
        }

        /**
        * Get the services provided by the provider.
        *
        * @return array
        */
        public function provides()
        {
            return ['redis', 'redis.connection'];
        }
    }
  ```
- 替换 RedisServiceProvider
  - 将 config/app.php 中的 providers 数组的 Illuminate\Redis\RedisServiceProvider::class 注释掉，然后将 App\Providers\ReplaceRedisServiceProvider::class 注册进 providers ，到此就完成了

#### 其他框架的实现
- jonahgeorge/jaeger-client-php 是一个通用的 client 包，但是 Hyperf 框架做了更加完善的封装，与 Laravel 不同的是，其 Redis 等部分是基于 AOP 来完成的，对代码侵入性非常低，有兴趣可以阅读一下 hyperf/tracer 的代码