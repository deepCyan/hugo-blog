+++
date = '2023-03-02T08:12:25+08:00'
draft = false
title = 'Golang Memery Boom'
summary = 'Go 程序的一次内存爆炸'
+++

几个月前写了一个非常简单的 web 程序，内容只有鉴权和提供用户信息，但是运行了一段时间后，内存居然炸掉了

工程除了 gin 并没有用任何的外部框架，几乎是裸写的，于是打开了 pprof 查看是哪里造成了内存无法释放

终于在获取用户信息的地方发现在使用 Redis 的时候每次都是新建一个链接，但是之后并没有释放

![pprof分析结果](/images/go-pprof.png)

redis.Connection 创建链接后，一直没有释放，每一次查询都会新建一个 Redis 链接，那么一定会内存越来越爆炸了

但是当时也是什么都不会（现在也是），直接加了一个 defer redis.Colse 就解决了问题，道理上还是应该建立连接池的