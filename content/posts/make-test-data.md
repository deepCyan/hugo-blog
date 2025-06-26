+++
date = '2025-03-06T19:23:20+08:00'
draft = false
title = '创建测试数据'
summary = '对 Laravel 的拙劣模仿'
+++

### 创建测试数据

#### 背景
一般来讲，在运行单元测试的时候，每一个 case 都应该拥有完全初始化的数据库状态，也就是空数据，然后由 case 自己制造数据，所以需要一个相对方便的制造测试数据的方式，但是 Hyperf 好想没有对应的组件，所以只好自己搞出来一个，这里采用的方案参考了 Laravel

#### 实现
- 在项目根目录新建 model_factory 目录，在 model_factory 中新建基类 ModelFactory，具体代码如下
  ```php
  <?php

    declare(strict_types=1);

    namespace ModelFactory;

    use Faker\Factory;
    use Hyperf\DbConnection\Model\Model;

    class ModelFactory
    {
        protected $model;

        /** @var Model */
        protected $modelIns;

        protected $faker;

        protected $loopNumber = 0;

        public function __construct()
        {
            $this->modelIns = new $this->model();
            $this->faker = Factory::create('zh_CN');
        }

        public function __call($name, $arguments)
        {
            return $this->modelIns->newQuery()->{$name}(...$arguments);
        }

        public function create(array $coverData = [])
        {
            return $this->modelIns->newQuery()->create(array_merge($this->definition(), $coverData));
        }

        public function many(int $number)
        {
            $this->loopNumber = $number;
            return $this;
        }

        public function createMany(array $coverData = [])
        {
            for ($i = 0; $i < $this->loopNumber; ++$i) {
                $this->modelIns->newQuery()->create(array_merge($this->definition(), $coverData));
            }
        }

        public function definition()
        {
            return [];
        }
    }
  ```
- 同 Laravel 一样，我们为每个 Model 设置其对应的 Factory 来创建数据，其配置也非常简单，举个🌰
  ```php
  <?php

    declare(strict_types=1);

    namespace ModelFactory\Config;

    use App\Model\User\UserTagConfig;
    use ModelFactory\ModelFactory;

    class UserTagConfigFactory extends ModelFactory
    {
        public $model = UserTagConfig::class;

        public function definition()
        {
            return [
                'tag_name' => $this->faker->title,
            ];
        }
    }
  ```
    如此，Model UserTagConfig 继承 ModelFactory，并设置成员属性 $model 指明 Model 类，然后重写 definition 方法，设置此 model 的字段值以用作生成，另外值的指出的是，由于基类已经设置了 Faker 类，所以在具体的 factory 中可以直接使用 this->faker 来创建虚拟数据

- 使用也非常简单，依旧使用具体的代码来举例
  ```php
    class UserTagTest extends BaseTest
    {
        public function testUserTag()
        {
            (new UserTagConfigFactory())->create();
            (new UserTagConfigFactory())->create([
                'other' => 'xxx',
            ]);
            (new UserTagConfigFactory())->many(20)->createMany();
            $service = ApplicationContext::getContainer()->get(UserTagService::class);
            $tagList = $service->fetchTagList();
            $this->assertEquals(20, count($tagList[0]['tags']));
            $service->setUserTag(1, 1, range(1, 10));
            $service->setUserTag(1, 2, range(5, 10));
            $service->setUserTag(1, 3, range(10, 15));
            $userTags = $service->fetchUserTags(1);
            $this->assertEquals(6, count($userTags[2]['tags']));
        }
    }
  ```
  基于一个简单的单测 case，我们可以很清晰的看出 Factory 是如何被使用的，直接调用 create 方法会使用 definition 中设置的值来创建数据，当然 create 支持传参后覆盖初始设置，many 方法可以指定需要创建的数据个数，只不过后续需要调用 createMany 方法来执行创建，当然和 create 方法一样，支持传参后覆盖初始设置

- 当然我们需要将 model_factory 目录纳入 composer 加载目录，对 composer.json 做如下修改，在 autoload-dev.psr-4 中增加配置项
  ```json
   "autoload-dev": {
        "psr-4": {
            "HyperfTest\\": "./test/",
            "ModelFactory\\": "./model_factory"
        }
    },
  ```