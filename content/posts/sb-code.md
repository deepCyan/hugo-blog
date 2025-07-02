+++
date = '2024-08-22T19:04:22+08:00'
draft = false
title = '气人的代码'
summary = '全是情绪, 没有输出'
+++

#### 引言
- 一直以来，已经有很多人论述了，如何正确的书写代码，而鲜有人对不那么合理的代码书写方式做一些分析，这就是本文的目的（也带一些个人情绪
- 这里仅仅讨论 PHP 的代码

#### 样例

- 四处飞舞的 static 方法，考虑如下代码
  ```php
    class Foo
    {
        public static function bar()
        {
            self::one();
            self::two();
            self::three();
        }
    }
  ```
  当然，这种代码可以正常运行，这没问题，但是 static 方法其实剥夺了类的性质，一个四处都是 static 的类，不再是一个对象的模版，而是一堆方法的聚集地，这是反 OOP 的，你无法在 static 中使用 this 来调用对象的上下文方法和属性，也难以进行依赖注入，总结来说，static 是面向过程而非面向对象的，如果你试图写的就是面向过程的代码，那其实没有什么问题，但是如果你使用了一个面向对象的框架，那应当遵循一些基础的开发约定

  其实 static 如何使用，是很清晰的，无非以下几种情况
  - 需要单例（使用容器来管理单例可能是更好的方法）
  - 方法本身是无状态的
  - 可能还有工厂模式，也可以使用 static 方法获取实例

- 使用 & 来操作数据，同样的，考虑以下代码
  ```php
    function update1(array &$params)
    {
        // do sth.
    }

    $demo = [];

    update1($demo);
    update1($demo);
    update1($demo);
  ```
    这种缺点是很明显的，将数据的修改隐藏在了方法内部，如果这种方法很多的话，甚至不需要很多，在某个不起眼的角落调用一次，就可能被人忽略，进而失去了对某个变量的管理

- 使用错误而不是异常，样例如下
  ```php
    $createGroup = $this->xxxService->createGroup([]);
    if ($createGroup === false) {
        // 这写法真逆天
        throw new \LogicException($this->xxxService->getError());
    }
    // 发一条消息, 激活群
    $pushMessage = $this->xxxService->sendGroupMsg([]);
    if ($pushMessage === false) {
        // SB
        throw new \LogicException($this->xxxService->getError());
    }
  ```
  首先需要承认，使用异常还是错误，在不同的业务场景各有优劣，不能直接说某个方法是不好的，比如 golang 就是这种处理方式，当然了，golang 也因此饱受诟病和调侃（啧

  但是需要明确的是，在 PHP 的 Web 场景下，尤其是使用传统的 Web 框架比如 Laravel 的情况下，这种方式是极不恰当的

  一、这种写法的缺陷之处不只是代码啰嗦，而是如果别人不知道要主动获取错误，那就会一直往下走，出现意料外的异常

  二、可以注意到这个代码将错误信息直接记录到了 Service 类上面，这样 Service 就变成了一个有状态的类，背离了我们设置 Service 层的初衷，同样的，这种代码在某些框架比如 Hyperf 是有很大的风险隐患的


- 变量风格的不统一和模糊的变量命名
  这两种情况都很好理解，变量风格上，无非是一会蛇形命名一会驼峰命名，模糊的变量命名，指的是一些形如 data / info / model ，没有指向性，可解释范围大到离谱的变量命名