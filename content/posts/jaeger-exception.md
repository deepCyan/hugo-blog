+++
date = '2023-04-19T08:08:58+08:00'
draft = false
title = 'Jaeger 导致的测试异常'
summary = 'Jaeger导致的测试异常'
+++

事情的起因是要给一个新的 hyperf2.2 项目集成单元测试

本来很简单，把以前的东西直接放进去就好了，最多增减几个 setup 的 mock 类

但是诡异的事情出现了

执行 compsoer test 出现这个错误

> Script co-phounit -prepend test/ootstrap,php -C phpunitxml --colors-almays --testdox handling the test event returmed with eror code 25

当然什么都看不出来，于是加个 -vvv 的参数看看，依旧没什么有用的信息甚至还抛出来了更离奇的异常（这个至今不知道怎么回事）

> pentl_fork() has been disabled for security reason in ...

于是继续打 var_dump ，我想到开始是要初始化容器的，是不是在构造某个类的时候出了问题，于是在容器的 make 方法中一个个打出来，看是哪个类出了问题，果然发现了异样，有问题的项目打印出来的类名远少于正常执行的对照项目（是的，我有一个可以 run 的项目），并且最后几个类名是 Tracer 相关的，看来问题出在 Tracer 上

然后我发现了和对照项目的差异，对照项目的默认 tracer driver 是 zipkin，而当前默认的是 jaeger

于是换成默认 zipkin，成功啦

可是，为什么呢，于是我一个个类追了进去，终于在 Jaeger\ThriftUdpTransport::doOpen 中找到了原因

```php
private function doOpen(): void
{
    $this->socket = @socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
    $ok = @socket_connect($this->socket, $this->host, $this->port);
    if ($ok === false) {
        throw new TTransportException('socket_connect failed');
    }
}
```

我发现这里的调用函数被抑制了，这是可以理解的，因为 Jaeger 只是一个监控，不应当影响主业务，但是这里的 @ 导致我无法得到任何信息，去掉 @ 后，再次执行，终于看到了清晰的报错

> PHP Fatal error:  Uncaught Swoole\Error: API must be called in the coroutine in @swoole-src/library/ext/sockets.php:21

很好，触及了知识盲区，不知道为什么会这样，但是可以解决，在 test/bootstrap.php 做如下修改

```php
! defined('SWOOLE_HOOK_FLAGS') && define('SWOOLE_HOOK_FLAGS', SWOOLE_HOOK_ALL ^ SWOOLE_HOOK_SOCKETS);
```

既然你要协程环境，那老子不 hook 你就是了

然后继续报错

> Uncaught Swoole\Error: API must be called in the coroutine ... gethostbyname

继续修改如下

```php
! defined('SWOOLE_HOOK_FLAGS') && define('SWOOLE_HOOK_FLAGS', SWOOLE_HOOK_ALL ^ SWOOLE_HOOK_SOCKETS ^ SWOOLE_HOOK_BLOCKING_FUNCTION);
```

摆平，此时默认是 jaeger 也可以正常开启测试

但是讲道理，这种情况最好是在 testing env 中给一个另外的值来处理，大可以不必死磕它