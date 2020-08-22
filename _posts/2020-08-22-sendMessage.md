---
layout: post
title: 消息发送
categories: RocketMQ
description: 
keywords: Java, RocketMQ, 消息发送
---
# 消息发送解析

Rocketmq源码中，在example模块下有很多例子，其中quickstart中有最简单的发送消息和消费消息的过程，我们以quickstart中的消息发送为例简单分析一下消息发送的过程

<img src="/images/posts/rocketmq/project-structure.png" width = "300" height = "500" align=center />

## 类图
与producer相关的逻辑在client模块下，实际上对于producer和consumer的逻辑都在client模块下。
<img src="/images/posts/rocketmq/client-producer.png" width = "500" height = "400" align=center />


## 使用

删掉部分冗余代码之后，客户端发送消息逻辑如下

```
        // new一个producer对象，指定producer group
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 首先调用start()
        producer.start();

        // 循环发送1000条消息
        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("TopicTest" /* Topic */,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
                );
                // 调用发消息逻辑
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        // 使用完之后，调用shutdown
        producer.shutdown();
```



## 剖析