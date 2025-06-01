+++
date = '2023-06-07T08:27:04+08:00'
draft = false
title = '做个小游戏-用户绑定'
summary = '其实用缓存也不是不行'
+++

加入房间要处理两件事
- 建立用户和房间的映射
- 建立用户和 fd 的映射

其实建立这两个映射很简单，比如很自然的就能想到数据库，不管是 mysql 还是 redis，但是这样会增大系统的复杂度，原因如下
- 用户的 fd 只在链接可用时有效，用户弱网环境应当考虑为一个常见场景，这必然会造成过多的 IO
- 链接只在服务可用的时候有意义，也就是 fd 只在服务可用的时候有意义，在服务挂掉 / 重启的时候所有用户和 fd 的绑定都应该失去意义

综上考虑，我们决定将用户和 fd 的映射关系存储在内存中，用户和房间的关系也一并存储在内存中并在适当的时候落盘

决定了如何建立映射关系，就还剩下一个问题悬而未决，就是系统如何描述房间和用户

让我们这么想，房间有什么东西？房间 id, 房间内用户等等，房间可以做什么？加入 / 踢出用户，同样的，用户有用户 id, 昵称，可以修改昵称等等

这么想就很清楚了，这俩就是个对象嘛，房间有什么，就是一个类的属性，能做什么，就是类的方法

好的，那就新建两个类，Room 和 User

- Room 类暂时构建如下
  ```php
    namespace App\Entity;

    class Room
    {
        protected int $room_id = 0;

        protected array $players = [];

        protected array $user_fds = [];

        public function getRoomId()
        {
            return $this->room_id;
        }

        public function setRoomId(int $roomId)
        {
            $this->room_id = $roomId;
        }

        public function setPlayer(int $userId)
        {
            $this->players[] = $userId;
        }

        public function removePlayer(int $userId)
        {
            foreach ($this->players as $index => $player) {
                if ($userId == $player) {
                    array_splice($this->players, $index, 1);
                }
            }
        }

        public function setUserFd(int $userId, int $fd)
        {
            $this->user_fds[$userId] = $fd;
        }

        public function getFds()
        {
            return array_values($this->user_fds);
        }
    }
  ```

- 同理，User 类暂时定义如下
  ```php
    namespace App\Entity;

    class User
    {
        protected int $user_id = 0;
        
        protected int $room_id = 0;
        
        protected int $fd = 0;
        
        protected string $nickname = '';
        
        public function getUserId()
        {
            return $this->user_id;
        }
        
        public function setUserId(int $userId)
        {
            $this->user_id = $userId;
        }
        
        public function getRoomId(int $roomId)
        {
            $this->room_id = $roomId;
        }
        
        public function setRoomId(int $roomId)
        {
            $this->room_id = $roomId;
        }
        
        public function setFd(int $fd)
        {
            $this->fd = $fd;
        }
        
        public function getFd()
        {
            return $this->fd;
        }
        
        public function setNickname(string $nickname)
        {
            $this->nickname = $nickname;
        }
        
        public function getNickname()
        {
            return $this->nickname;
        }
    }
  ```

描述了用户和房间这两个实体，下一步就是建立 RoomManager 和 UserManager 来管理这两个东西，代码如下
- RoomManager
  ```php
    namespace App\Provider;

    use App\Entity\Room;
    use Hyperf\Logger\LoggerFactory;
    use Psr\Log\LoggerInterface;

    class RoomManager
    {
        protected LoggerInterface $logger;
        protected array $rooms;

        public function __construct(LoggerFactory $loggerFactory)
        {
            $this->logger = $loggerFactory->get('room_manager');
        }

        public function hasRoom(int $roomId): bool
        {
            return isset($this->rooms[$roomId]);
        }

        public function getRoom(int $roomId): Room
        {
            if (! $this->hasRoom($roomId)) {
                throw new \Exception('room not found: ' . $roomId);
            }
            return $this->rooms[$roomId];
        }

        public function addRoom(Room $room)
        {
            $this->rooms[$room->getRoomId()] = $room;
        }

        public function fetchRooms(): array
        {
            return $this->rooms;
        }
    }
  ```

- UserManager
  ```php
    namespace App\Provider;

    use App\Entity\User;

    class UserManager
    {
        protected array $users = [];

        protected array $fdUser = [];

        public function setUser(User $user)
        {
            $this->users[$user->getUserId()] = $user;
            $this->fdUser[$user->getFd()] = $user;
        }

        public function getUser(int $uid): User
        {
            return $this->users[$uid];
        }

        public function hasUser(int $uid)
        {
            return isset($this->users[$uid]);
        }

        public function getUserFromFd(int $fd): User
        {
            return $this->fdUser[$fd];
        }

        public function removeFd(int $fd)
        {
            unset($this->fdUser[$fd]);
        }

        public function removeUser(int $uid)
        {
            unset($this->users[$uid]);
        }

    }
  ```

- 那么做完了这些工作，用户加入房间这个操作就变得非常简单了，只需要设置房间，用户，fd 三者的绑定关系即可，我们可以新建一个 JoinRoomService 来完成这件事，代码如下
  ```php
    namespace App\Service;

    use App\Entity\Room;
    use App\Entity\User;
    use App\Provider\RoomManager;
    use App\Provider\UserManager;
    use Hyperf\Logger\LoggerFactory;
    use Psr\Log\LoggerInterface;
    use Room\JoinRoomRequest;

    class JoinRoomService
    {
        protected LoggerInterface $logger;

        protected RoomManager $roomManager;

        protected UserManager $userManager;

        public function __construct(LoggerFactory $loggerFactory, RoomManager $roomManager, UserManager $userManager)
        {
            $this->logger = $loggerFactory->get('join_room');
            $this->roomManager = $roomManager;
            $this->userManager = $userManager;
        }


        public function joinRoom(JoinRoomRequest $joinRoomRequest, int $fd)
        {
            $roomId = $joinRoomRequest->getRid();
            $userId = $joinRoomRequest->getUserId();
            if ($this->roomManager->hasRoom($roomId)) {
                $room = $this->roomManager->getRoom($roomId);
            } else {
                $room = new Room();
                $room->setRoomId($roomId);
                $this->roomManager->addRoom($room);
            }
            $room->setUserFd($userId, $fd);
            if ($this->userManager->hasUser($userId)) {
                $user = $this->userManager->getUser($userId);
            } else {
                $user = new User();
                $user->setUserId($userId);
            }
            $user->setFd($fd);
            $this->userManager->setUser($user);
        }
    }
  ```

- 剩下的工作就只是将这些业务代码组装起来了，修改 DispatchMessage::dispatch ，增加一个 case
  ```php
    namespace App\Provider;

    use App\Service\JoinRoomService;
    use Base\BaseData;
    use Hyperf\Utils\ApplicationContext;

    class DispatchMessage
    {
        public function dispatch(BaseData $baseData, int $fd)
        {
            $container = ApplicationContext::getContainer();
            switch ($baseData->getBody()) {
                case 'join_room_request':
                    return $container->get(JoinRoomService::class)->joinRoom($baseData->getJoinRoomRequest(), $fd);
                default:
                    throw new \Exception('Invalid body', 400);
            }
        }
    }
  ```