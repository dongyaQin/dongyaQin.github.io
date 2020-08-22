---
layout: post
title: 消息发送
categories: RocketMQ
description: 
keywords: Java, RocketMQ, 消息发送
---
# 消息发送解析

Rocketmq源码中，在example模块下有很多例子，我们以一个消息发送具体的例子来开始解析消息发送过程

## 基本结构

网络模块在包*remoting*中，5000多行代码，为rocketmq各个组件之间的网络通信提供基本功能

从接口层面，client和server的相关的类最终会分别实现RemotingClient和RemotingServer接口
![](/images/posts/rocketmq/remotting-interface.png)
