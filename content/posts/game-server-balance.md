+++
date = '2023-06-05T08:12:17+08:00'
draft = false
title = '游戏服务器分流'
summary = '这个是对游戏服务器分流的一个简易解决方案'
+++

### 背景

之前做了一些游戏，技术方案是 Hyperf （Swoole）的单一 worker 模式来做的逻辑处理（为了便于维护，使用的对象而非 Table），这样可以将数据在内存中处理，大幅度提高响应速度，但是会造成 CPU 资源的浪费，故而需要多开几个 Server，关于如何维护服务数据，以后可以单开一篇来说明

### 为什么要分流

你说呢

### 分流的策略

这个有很多种选择，由于我的游戏是基于【房间】或者【群组】的，所以我的想法是将用户依据用户所处的群组来分流，群组 A 的用户应当对于群组 B 的事情一无所知

### 分流的工具

NGINX 的 hash 分流，配置文件如下

```nginx
upstream game_nodes {
    hash $http_gid;
    server ip1:port1;
    server ip2:port2;
}

server {
    listen 80;
    server_name domains;
    location / {
        proxy_set_header Upgrade "websocket";
        proxy_set_headerConnection "upgrade"
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_http_versions 1.1;
        # 转发到多个 ws server
        proxy_pass http://game_nodes;
    }
}
```
围绕这个配置文件来说明一下分流的方案

- 首先在客户端的请求头中增加一个 gid, 也就是用户所处的群组 id
- 服务端开启 N 个服务，也即 ip1:port1, ip2:port2...
- 设置 upstream 并指定分流策略为依据 gid 的值进行 hash 分流以期群组在节点分布均匀（注意，这并不能保证压力均匀）
  - upstream.hash 配置项目表示使用一致性 hash 来分发流量
  - $http_gid 表示从请求头中获取 gid 字段
- 在 server 配置项中使用反向代理将请求转发到上文设置的 upstream ，也即 game_nodes

### 分流的问题以及解决方法
因为游戏服务通信量是很大的，所以在某个节点重启中这个节点的负载会被分摊去其他节点，这个过程可能会出现群组不一致的问题

对于这个情况，有一个不成熟的方案，首先关闭 nginx ，使得客户端处于断网重连状态，然后重启所有节点后开启 nginx 响应