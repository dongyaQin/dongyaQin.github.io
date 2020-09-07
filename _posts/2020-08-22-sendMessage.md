---
layout: post
title: 消息发送
categories: RocketMQ
description: 
keywords: Java, RocketMQ, 消息发送
---
# 消息发送解析


## 类图
与producer相关的逻辑在client模块下。在rocketmq里，相对于broker和nameserver，对于producer和consumer都是client。
<img src="/images/posts/rocketmq/client-producer.png" width = "500" height = "400" align="middle" />


### ClientConfig

ClientConfig顾名思义，是负责描述client配置的一个类，例如：

```
    // nameserver地址
    private String namesrvAddr = NameServerAddressUtils.getNameServerAddresses();
    private String clientIP = RemotingUtil.getLocalAddress();
    private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");
    private int clientCallbackExecutorThreads = Runtime.getRuntime().availableProcessors();
    protected String namespace;
    protected AccessChannel accessChannel = AccessChannel.LOCAL;
    private int pollNameServerInterval = 1000 * 30;
    private int heartbeatBrokerInterval = 1000 * 30;
    ...
```
由clientIP + instanceName + unitName唯一标识一个client，生成clientId。ClientConfig还配置了一些默认的interval配置，例如，从nameserver更新topic route info的间隔时间；往brocker发heartbeat的间隔时间等

### MQProducer

接口MQProducer主要定义了各种send方法，下面列出的三种同步send方式，也有相应的异步和单向的版本，对于同步和异步请求，还有可以设置请求timeout的版本

* SendResult send(final Message msg) : 同步send
* SendResult send(final Message msg, final MessageQueue mq): 选定MessageQueue send
* SendResult send(final Message msg, final MessageQueueSelector selector, final Object arg): 根据selector send

在版本4.6.0，rocketmq支持了Request-Response模型，具体可参考: [RIP16](https://github.com/apache/rocketmq/wiki/RIP-16-RocketMQ-RPC(Request-Response-model)-Support)，本文先不作讨论

### DefaultMQProducer & TransactionMQProducer

用户发消息使用DefaultMQProducer的对象，而TransactionMQProducer是发送事物消息要使用的类

## 剖析消息发送流程

接下来我们跟着代码剖析一下消息发送过程。Rocketmq源码中，在example模块下有很多例子，其中quickstart中有最简单的发送消息和消费消息的过程，我们以quickstart中的消息发送为例简单分析一下消息发送的过程。删掉部分冗余代码之后，客户端发送消息逻辑如下

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

### producer.start()

DefaultMQProducer是曝露给用户使用的类，其内部主要调用了DefaultMQProducerImpl实现各种sendMessage的方法。调用DefaultMQProducer的start()时，实际调用了DefaultMQProducerImpl的start()，在DefaultMQProducerImpl的start()中，比较重要的是从MQClientManager通过工厂方法得到MQClientInstance的实例，然后调用其start(), 默认情况下，每个进程共享一个MQClientInstance实例，MQClientInstance实例很重要，其内部维护了producerTable， consumerTable， topicRouteTable等基本信息，内部使用其他组件跟broker, nameserver通信等。以下为MQClientInstance的start()实现：

```
        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    this.startScheduledTask();
                    // Start pull service
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
```
其中:

* this.mQClientAPIImpl

MQClientAPIImpl的实例，主要负责跟broker和nameserver的通信，包括sendMessage, getTopicRouteInfoFromNameServer等等，this.mQClientAPIImpl.start()只有行代码，那就是调用RemotingClient的start()，在前一篇文章中已经分析

* this.startScheduledTask()

 开启一系列的定时任务，其中包括

	1. 每隔两分钟通过this.mQClientAPIImpl获取nameServer地址fetchNameServerAddr()
	2. 每隔30s对MQClientInstance中所有consumer subscribe的topic以及producer publish的topic进行updateRouteInfo
	3. 每隔30s删除offline的broker，以及通过this.mQClientAPIImpl往所有的broker发送HeartbeatData， HeartbeatData中包含了此client所有的producer信息和consumer信息
	4. 每隔5s persist所有consumer的offset
	5. 每隔一分钟更改consumer的ThreadPool的CorePoolSize，不过这个代码目前被注掉了

* this.pullMessageService.start();

PullMessageService继承自ServiceThread，本身是一个线程实例，启动之后，循环的从pullRequestQueue(LinkedBlockingQueue)中取出pullRequest，然后根据pullRequest的consumerGroup获取到对应的consumer，然后执行consumer的pullMessage逻辑。那什么时候是谁往pullRequestQueue中发消息，且看下面的rebalance

```
    @Override
    public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                PullRequest pullRequest = this.pullRequestQueue.take();
                this.pullMessage(pullRequest);
            } catch (InterruptedException ignored) {
            } catch (Exception e) {
                log.error("Pull Message Service Run Method exception", e);
            }
        }

        log.info(this.getServiceName() + " service end");
    }
```

* this.rebalanceService.start();

RebalanceService也是继承自ServiceThread，start()之后，循环调用MQClientInstance的doRebalance()方法，MQClientInstance进而再调用所有consumer的doRebalance()方法

至此，producer.start()的准备工作完成，下面看下发送消息的具体流程


### producer.send(msg)

producer.send(msg)作为同步且只有一个参数Message的方法，最终调用了下面这个sendDefaultImpl方法。包括异步和单向调用的默认实现也是这个，此方法我省略了一些非重要的代码。

通过对sendDefaultImpl的代码分析，此方法主要是通过Topic找到TopicPublishInfo，进而select出要发送的MessageQueue，提供重试的功能，调用sendKernelImpl方法真正发送消息。

```
    private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // validate step
        ...
        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            	//重试次数
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            String[] brokersSent = new String[timesTotal];
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                //选取一个MessageQueue
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        if (times > 0) {
                            //Reset topic with namespace during resend.
                            msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                        }
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                        //所有重试时间加总计算是否超时
                        if (timeout < costTime) {
                            callTimeout = true;
                            break;
                        }
                        // internal send
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        switch (communicationMode) {
                            case ASYNC:
                            case ONEWAY:
                                return null;
                            case SYNC:
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }

                                return sendResult;
                            default:
                                break;
                        }
                    } catch (RemotingException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        exception = e;
                        continue;
                    } catch(...){
                    	...
                    }
              } else {
                    break;
             }
            }

            if (sendResult != null) {
                return sendResult;
            }
            ...
            if (callTimeout) {
                throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
            }

		...

            throw mqClientException;
        }

        validateNameServerSetting();

        throw new MQClientException("No route info of this topic: " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
            null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
    }
```

sendKernelImpl方法主要是

1. 根据MessageQueue找到broker的addr
2. 如果有MessageHook，执行
3. 构建SendMessageRequestHeader，设置基本信息，比如ProducerGroup，Topic，Flag等信息
4. 调用MQClientAPIImpl的sendMessage方法发送消息

MQClientAPIImpl的sendMessage主要就是根据发送消息的交互类型(同步，异步，单向)调用remotingClient的对应方法发送了

## 总结

本文主要阐述发送消息的粗略流程，一些细节并未详细阐述，例如，

* 如何选择MessageQueue发送
* 获取TopicPublishInfo流程是怎么样的
* 异步消息回调处理流程

这些问题在后边再补充，或者在其他章节中再细谈