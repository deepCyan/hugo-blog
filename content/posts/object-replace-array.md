+++
date = '2023-02-16T08:09:24+08:00'
draft = false
title = '为什么要引入对象来替代数组'
summary = '省流版本：对象比数组读起来方便'
+++

其实这个问题已经有了极为广泛的讨论，也有了相对明确的答案——尽可能使用对象，但是为了以后和人争论有充足的草稿，我要将这件事再次论述一次

首先要明确的是，下文讨论的数组，都是关联数组（map）而非索引数组（list）

PHP 的数组是一个极其优秀的设计，一个 value 可以存储任意类型的东西，并且提供了大量的 array 函数，给予了开发者非常宽松的开发条件，但是与之而来的问题就是，代码变得难以阅读和理解，尤其在调用链路过长的时候，更是让人挠头，还是从具体的代码入手

考虑这个函数

```php
<?php
function show(array $data)
{
    foreach ($data as $key => $value) {
        echo $key . ': ' . $value;
    }
    return;
}

function run(array $data)
{
    show($data);
}

function start(array $data)
{
    run($data);
}
```

很明显，我们无法得知 data 内部究竟有什么，虽然在开发时非常快速（可能这也是 PHP 效率高的一部分原因），但是如果一个月后来回看的话，这是非常让人费解的代码，如果没有注释的话，我们可能需要一步步 var_dump 才能得知 data 的信息，这显然是令人不爽的

如果说难以阅读是一个重要的问题，那么下一个问题就更加致命，—— 我们无法保证一直书写正确的 key name，这其实是需要 IDE 的帮助的，但是显然的，IDE 并不能分析出这个数组内部有什么东西，毕竟连开发者都需要打印来看看

那么如果用对象的话，可以怎么处理呢

```php
class MyData
{
    protected $name;

    protected $address;

    public function getName()
    {
        return $this->name;
    }

    public function setName(string $name)
    {
        $this->name = $name;
    }

    public function getAddress()
    {
        return $this->address;
    }

    public function setAddress(string $address)
    {
        $this->address = $address;
    }

    public function show()
    {
        echo 'name' . $this->name;
        echo 'address' . $this->address;
    }
}

function show(MyData $data)
{
    $data->show();
}
```

是的，代码量增加了很多，但是我们可以明确得知 $data 的内部数据，并且提供了安全的类型检查，在 IDE 的帮助下可以彻底避免错误的 key name

事实上，仅仅考虑这种数组传参的情况，也就是 DTO （data to object），不同框架已经给出了各自的解决方案，如 Laravel 和 Hyperf 就提供了 Fluent 类

在结尾我也提供一个适用于 Laravel 和 Hyperf 的 DTO 基类

```php
<?php
class Format
{
    public function __construct(array $data = [])
    {
        foreach ($data as $key => $value) {
            $method = Str::camel('set_' . $key);
            $this->{$method}($value);
        }
    }

    public function __call($name, $args)
    {
        if (substr($name, 0, 3) == 'get') {
            $field = Str::snake(substr($name, 3));

            return $this->{$field} ?? null;
        }

        if (substr($name, 0, 3) == 'set') {
            $field = Str::snake(substr($name, 3));
            if (property_exists($this, $field)) {
                $this->{$field} = $args[0] ?? null;
            }

            return $this;
        }

        throw new \Exception('call to undefined method: ' . $name);
    }

    public function toArray()
    {
        $formatArray = [];
        $allProtectedFields = $this->getAllProtected();
        foreach ($allProtectedFields as $k => $field) {
            $formatArray[$field] = $this->{$field};
        }
        return $formatArray;
    }

    public function toArrayNotNull()
    {
        $formatArray = [];
        $allProtectedFields = $this->getAllProtected();
        foreach ($allProtectedFields as $k => $field) {
            $v = $this->{$field};
            if (! is_null($v)) {
                $formatArray[$field] = $v;
            }
        }
        return $formatArray;
    }

    public function getAllProtected()
    {
        $ref = new \ReflectionClass($this);
        $propers = $ref->getProperties();
        $res = [];
        foreach ($propers as $key => $value) {
            /* @var $value \ReflectionProperty */
            if ($value->getModifiers() == $value::IS_PROTECTED) {
                $res[] = $value->name;
            }
        }
        return $res;
    }
}
```

举个例子

```php
/**
 * @method getName()
 * @method getType()
 * @method getLength()
 * @method getDefault()
 * @method getComment()
 * @method getNotNull()
 *
 * @method setName($value)
 * @method setType($value)
 * @method setLength($value)
 * @method setDefault($value)
 * @method setComment($value)
 * @method setNotNull($value)
 */
class AnalyseTableColumnFormat extends Format
{
    protected $name;

    protected $type;

    protected $length;

    protected $default;

    protected $comment;

    protected $not_null;
}
```

是的，这个基类只需要定义 protected 的成员属性即可，Format 类已经实现了 getter 和 setter（当然可以自己重写）

但是为了对 IDE 友好，我们还是要加上类注释，对此我也提供了一个自动生成注释的脚本，但是代码写的比较简陋且通用性并不强，就不放出来了...