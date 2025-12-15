+++
date = '2025-12-15T18:55:16+08:00'
draft = false
title = 'Swoole Cmath'
summary = 'Swoole 编译出错 cmath 的问题'
+++

### Swoole 编译出现 cmath 错误
mac 环境编译出现 fatal error:'cmath'file not found，寻找半天问题，记录一下解决方法

临时设置环境变量，再次 make 即可

export C_INCLUDE_PATH=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include
export CPLUS_INCLUDE_PATH=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1