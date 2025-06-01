+++
date = '2023-06-06T08:18:07+08:00'
draft = false
title = '做个小游戏——准备篇'
summary = '闲着也是闲着'
+++

### 目的

详细记录一下如何使用 Hyperf 构建一个简单的不能更简单的游戏——石头剪刀布（好像复杂的用 PHP 确实不太好）

### 仓库地址
- 业务仓库 https://gitee.com/wang-zhihui-release/stone_game
- PB 仓库 https://gitee.com/wang-zhihui-release/stone_game_pb

### 采用数据传输方式
webSocket 是双全工的协议,不论在 client 还是 server 都需要 onMessage 回调来处理信息, 那么 onMessage 方法很自然就成为了程序的入口, 但是有一个新的问题: server-client 之间通信的信息格式是什么, 当然, 很自然可以想到 json, 但是这里我要使用 Protobuffer, 为什么选择 PB 而不是 json? 因为 json 解析出来是一个关联数组, 而 PB 可以轻易的转为对象, 在以前的文章中我们讨论过为什么不要使用数组, 在这个场景下对象是非常合适的, 这一点在后续的代码编写中会十分明显,当然了, 如果不了解 PB 建议先百度一下 PB 相关的基础知识, 还是很简单的

### 步骤
- 首先新建一个工程 `composer create-project hyperf/hyperf-skeleton stone_game`
- 引入 ws-server `composer require hyperf/websocket-server`
- 增加 App\Controller\ServerController
  ```php
    namespace App\Controller;

    use Hyperf\Contract\OnCloseInterface;
    use Hyperf\Contract\OnMessageInterface;
    use Hyperf\Contract\OnOpenInterface;
    use Swoole\Http\Request;
    use Swoole\WebSocket\Frame;

    class ServerController implements OnMessageInterface, OnOpenInterface, OnCloseInterface
    {
        public function onMessage($server, Frame $frame): void
        {
        }

        public function onOpen($server, Request $request): void
        {
        }

        public function onClose($server, int $fd, int $reactorId): void
        {
        }
    }
  ```
- 然后根据文档说明，增加 ws 配置和路由，我的路由是这样的
  ```php
    Router::addServer('ws', function () {
        Router::get('/ws', App\Controller\ServerController::class);
    });
  ```

- 现在暂时放一下业务仓库, 我们新建一个新的 PB 仓库并将其做成 composer 包以进行内部引用, 如何打包这里不进行赘述, 只将关注点聚焦在 PB 文件中

- 简单新建两个 pb 文件, base.proto 和 room.proto, 内容如下, 刚刚开始, 这里我们只简单定义一个加入房间的请求消息和一个广播消息

- base.proto
  ```proto
    syntax = "proto3";

    package Base;

    import "room.proto";

    message Data {
        int64 server_time = 1;
        int32 version = 2;
        int64 rid = 3;
        string error_message = 4;
        oneof body {
            Room.JoinRoomRequest join_room_request = 100;

            Room.JoinRoomBroad join_room_broad = 500;
        }
    }
  ```
- room.proto
  ```proto
    syntax = "proto3";

    package Room;

    message JoinRoomRequest {
        int32 rid = 1;
        int32 user_id = 2;
    }

    message JoinRoomBroad {
        int32 rid = 1;
        int32 user_id = 2;
    }
  ```

- 引入仓库, 修改 composer.json
  ```json
  {
    "repositories": {
        "yunqing/stone_game_pb": {
            "type": "vcs",
            "url": "git@gitee.com:wang-zhihui-release/stone_game_pb.git"
        }
    },
    "require": {
        "yunqing/stone_game_pb": "~0.0.1"
    }
  }
  ```
- pb 仓库就绪, 继续回到 onMessage 中, 首先处理一下加入房间, 实际这个方法非常重要, 即便是最简单的情况下, 我们也需要思考几个问题
  - 如何进行消息的分发（加入房间就是一个消息，或者说事件）
  - 如何进行用户 id 和房间的绑定
  - 如何进行用户 id 和连接 fd 的绑定
- 首先解决一下消息分发的问题，当然你也可以把所有分发写在 ServerController 中，只是代码很难看而已
- 在 App\Provider 中新建 class DispatchMessage，基本结构如下
  ```php
    namespace App\Provider;

    use Base\BaseData;

    class DispatchMessage
    {
        public function dispatch(BaseData $baseData, int $fd)
        {
            switch ($baseData->getBody()) {
                default:
                    throw new \Exception('Invalid body', 400);
            }
        }
    }
  ```

- 依赖这个类，我们就可以将不同的消息转发给不同的服务，只需要在 ServerController 调用它即可
  ```php
    protected $disPatchMessage;

    public function __construct(LoggerFactory $loggerFactory, DispatchMessage $disPatchMessage)
    {
        $this->logger = $loggerFactory->get('server_controller');
        $this->disPatchMessage = $disPatchMessage;
    }

    public function onMessage($server, Frame $frame): void
    {
        $baseData = new BaseData();
        $res = base64_decode($frame->data);
        $baseData->mergeFromString((string) $res);
        $this->logger->info('request message', json_decode($baseData->serializeToJsonString(), true));
        $this->disPatchMessage->dispatch($baseData, $frame->fd);
    }
  ```
- 累了, 剩下的以后再写