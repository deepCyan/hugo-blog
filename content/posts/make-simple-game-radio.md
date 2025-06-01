+++
date = '2025-06-10T08:32:25+08:00'
draft = false
title = '做个小游戏-房间广播'
summary = '冲冲冲'
+++

### 发送消息的两种方式
- onMessage 的 Server 参数的 push 方法
- Hyperf\WebSocketServer\Sender::push 方法
- 两个方法的方法签名略有区别，但是前两个参数都是 fd , data，用起来是一样的，为了方便，一般用第二个

### 发送消息的参数来源
- fd, 也就是在 JoinRoom 时维护的 fd
- data, 要发送的消息, 自定义

### 咋个群发咯
- for，尽情的 for

### 封装一个广播类
- 我们可以封装一个广播类来处理各种消息的发送，包括单发，群发，指定消息，房间消息等等，大概可以这样
  ```php
    namespace App\Service;

    use App\Entity\Room;
    use Base\BaseData;
    use Hyperf\Logger\LoggerFactory;
    use Hyperf\Utils\Coroutine\Concurrent;
    use Hyperf\WebSocketServer\Sender;
    use Psr\Log\LoggerInterface;
    use Room\JoinRoomBroad;

    class RadioService
    {
        protected LoggerInterface $logger;

        protected Sender $sender;

        protected Concurrent $concurrent;

        public function __construct(
            LoggerFactory $loggerFactory,
            Sender $sender
        ) {
            $this->logger = $loggerFactory->get('radio');
            $this->sender = $sender;
            $this->concurrent = new Concurrent(10);
        }

        public function radioJoinRoom(JoinRoomBroad $joinRoomBroad, Room $room)
        {
            $base = new BaseData();
            $base->setJoinRoomBroad($joinRoomBroad);
            $this->radioRoom($room, $base);
        }

        private function radioRoom(Room $room, BaseData $base)
        {
            $res = base64_encode($base->serializeToString());
            $userFds = $room->getFds();
            foreach ($userFds as $userFd) {
                $this->concurrent->create(function () use ($userFd, $res) {
                    $this->sender->push($userFd, $res);
                });
            }
        }
    }
  ```

- 那么就可以在加入房间成功的时候发送一个 JoinRoom 的房间广播，以告诉房间内所有人有新人进房