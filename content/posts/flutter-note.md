+++
date = '2023-07-19T08:35:34+08:00'
draft = false
title = 'Flutter 笔记'
summary = '好像也用不太到'
+++

- GridView 页面尺寸崩溃，一直提示 pixel over..什么的，模拟器出现黄色警告条，解决方法是设置 GirdView.count 中 childAspectRatio 的参数，需要手动试出来

- 默认加载，不能在 initState 函数中使用 await 来进行修饰，一番搜索的解决方法是在 initState 函数中使用 Future.microtask 方法来做默认加载

- 文件上传，使用的是腾讯的 Cos 的 flutter 插件，报用，我的场景是在上传完成后获取地址然后进行提交，但是它的 SDK 并不提供这种方法，它的成功是一个回调，但是很明显我不能将完整的提交逻辑写在文件上传成功的回调里，后来在 G 站找了一个 tencent_cos 的组件，直接使用了 HTTP 接口上传(和服务端逻辑差不多)，还是只需要服务端生成临时的 ticket 即可

- 增减包后热更似乎失效，需要重新 run 一下

- Navigator 需要在...什么上下文中才可以使用，最开始我生成列表数据是直接写了一个函数以返回 Widget，但是这样植入 Navigator 后跳转会不可用，于是改成一个 class GetList extends StatelessWidget , 讲函数内容放在 build 方法中即可

- iOS 打包后无法在模拟器中运行了，似乎是打包的时候使用了 release 模式，但是开发时使用的是 debug 模式，改回来就好了，反正就...打包时候 xcode 改了什么东西，开发的时候就要改回来

- 但是打 release 包也有时间限制，似乎一周之后也就停用了，需要重新 build，很烦，垃圾 Apple
