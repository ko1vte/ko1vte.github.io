---
title: "gin-chat-uwu-糊出来的一个IM项目-第一期"
slug: "gin-chat-uwu-ep1"
date: 2023-06-05T20:51:48+08:00
tags: ["Go", "IM", "redis", "mysql", "websocket", "gin"]
aliases:
  - /SU15VTE/13f25f61.html
  - /SU15VTE/13f25f61
image: img/daliy.jpg
---

# 导言

在本学期，我用golang完成了几个《网络安全原理》的实验，不过都是用简单的代码实现了加密算法，在完成所有的实验后。我决定用golang来实现一个有关网络编程项目。我选择了这个IM项目，是因为我在很久之前就想写一个能让我和我的朋友一起聊天的聊天室。至于这个项目为什么叫 **gin-chat-uwu** 这是因为我对Real World CTF 2023的一道web题 **ChatUWU** 印象很深。~~一道题做不出来，坐牢一整天的痛苦记忆~~我实现这个项目后发现我的模板还是挺乱的，可以把代码写得可读性再强一些。

# 项目的规划

## 实现路线

我们的目标是实现一个简单的IM项目，而我选择先实现用户的注册，登陆，修改密码等功能，再实现一个websocket服务器，来供我们的用户聊天交互。

## tree

``` bash
.
├── Dockerfile      //docker构建文件
├── run.sh          //让项目在docker中以容器的形式运行
└── src
    ├── config
    │   └── config.yaml //包含mysql和redis的配置文件
    ├── dao
    │   ├── mysql.go    //mysql的CRUD操作
    │   └── redis.go    //redis的相关操作
    ├── database
    │   └── initdb.go   //初始化mysql和redis的连接
    ├── go.mod          //module
    ├── go.sum          //依赖
    ├── main.go         //main函数
    ├── middlewares     //中间件目录
    │   └── jwt.go      //jwt中间件的实现
    ├── models          //模型
    │   ├── message.go  //消息模型
    │   └── users.go    //用户模型
    ├── router          //路由
    │   └── router.go   //初始化的路由
    ├── services        //各项服务
    │   ├── ChatServer.go   //聊天功能的实现
    │   ├── DeleteServer.go //注销用户的实现
    │   ├── GetMessage.go   //从redis中获取聊天记录
    │   ├── LoginSever.go   //登陆功能的实现
    │   ├── RegisterSever.go    //注册功能的实现
    │   └── UpdateServer.go     //修改密码功能的实现
    ├── test                    //测试文件夹
    └── utils  
        │   ├── crypto.go   //里面包含散列函数             
        │   ├── readconf.go //实现了读取config的功能
```
