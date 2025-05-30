+++
date = '2023-04-01T08:33:57+08:00'
draft = false
title = '请求参数过多的bug'
summary = '请求参数过多的bug-php-swoole'
+++

背景是这样的

公司框架用的是 Hyperf ，前脚上了一个功能，后面就报错了

Bug 的报错是这样的

ErrorException: Swoole\Server::start(): Input variables exceeded 1000. To increase the limit change max_input_vars in php.ini.

报错很明显嘛，请求参数个数超了 PHP 异常导致 Swoole Woker 异常重启

但是就是找不到是哪个接口异常，而且客户端表示不可能存在请求 1000 个参数的接口

于是我想了一个办法，去切了 Hyperf\HttpServer\Server::onRequest 方法，切面的内容很简单，大概这样

```php
public function process(ProceedingJoinPoint $proceedingJoinPoint)
{
    $arguments = $proceedingJoinPoint->arguments['keys'];
    /** @var Request $request */
    $request = $arguments['request'];
    $psr7Request = Psr7Request::loadFromSwooleRequest($request);
    if (is_array($psr7Request->getParsedBody()) && count($psr7Request->getParsedBody()) > 900) {
        // log...
    }
    if (count($psr7Request->getQueryParams()) > 900) {
        // log...
    }
    if (count($psr7Request->getCookieParams()) > 900) {
        // log...
    }
    return $proceedingJoinPoint->process();
}
```

目的很简单，将参数过多的接口信息打印出来以排查问题，但是更诡异的事情出现了，报错在持续，但是 log 并没有任何内容

在思考很久后，我想到会不会是 PSR7 的请求参数和 PHP 认为的参数格式不太一样？

于是我从最原始的 request 开始处理，新的切面这个样子

```php
$arguments = $proceedingJoinPoint->arguments['keys'];
/** @var Request $request */
$request = $arguments['request'];
if (is_string($request->getContent())) {
    if (strlen($request->getContent()) > 3000) {
        $server = $request->server;
        $file = fopen('./debug.log', 'a+');
        fwrite($file, json_encode([
            'time' => date('Y-m-d H:i:s'),
            'path' => $server['request_uri'] ?? '',
            'body' => $request->getContent(),
        ]));
        fwrite($file, PHP_EOL . '-----------------------------------' . PHP_EOL);
        fclose($file);
    }
}
```

果然，这里精准打印到了 log

问题原因是因为客户端上传数组数据使用的是 param[]=1&param[]=2&param[]=3... 这种格式，通过 loadFromSwooleRequest 转化为 PSR7 请求对象的话，这是一个数组类型的参数，数组的内容就是 [1, 2, 3...]，但是原始的 PHP 脚本认为这就是 N 个参数，所以直接抛出了异常

这件事告诉我们，在用二分法排 bug 以前，要多想