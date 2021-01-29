# RocketMQ Store

## MQ 请求处理流程

```java
NettyDecoder ->
    NettyServerHandler -> channelRead0 -> processMessageReceived -> processRequestCommand
        -> NettyRemotingAbstract -> ExecutorService
            -> AsyncNettyRequestProcessor -> asyncProcessRequest
                -> SendMessageProcessor -> asyncProcessRequest -> asyncSendMessage
                    -> DefaultMessageStore#asyncPutMessage
                        -> CommitLog#asyncPutMessage
                            -> MappedFile#appendMessage
```

上面的流程的核心思想就是把网络请求通过 `ExecutorService` 进行异步化处理。

![rocket-netty-async.png](./images/rocket-netty-async.png)

## Index File

## store dir

abort
checkpoint
commitlog
config
consumequeue
index  
lock

```sh
# 在 store 目录下面执行
tree
```

输出

```sh
.
├── abort
├── checkpoint
├── commitlog
│   └── 00000000000000000000
├── config
│   ├── consumerFilter.json
│   ├── consumerFilter.json.bak
│   ├── consumerOffset.json
│   ├── consumerOffset.json.bak
│   ├── delayOffset.json
│   ├── delayOffset.json.bak
│   ├── subscriptionGroup.json
│   ├── subscriptionGroup.json.bak
│   ├── topics.json
│   └── topics.json.bak
├── consumequeue
│   └── TopicTest
│       ├── 0
│       │   └── 00000000000000000000
│       ├── 1
│       │   └── 00000000000000000000
│       ├── 10
│       │   └── 00000000000000000000
│       ├── 11
│       │   └── 00000000000000000000
│       ├── 12
│       │   └── 00000000000000000000
│       ├── 13
│       │   └── 00000000000000000000
│       ├── 14
│       │   └── 00000000000000000000
│       ├── 15
│       │   └── 00000000000000000000
│       ├── 2
│       │   └── 00000000000000000000
│       ├── 3
│       │   └── 00000000000000000000
│       ├── 4
│       │   └── 00000000000000000000
│       ├── 5
│       │   └── 00000000000000000000
│       ├── 6
│       │   └── 00000000000000000000
│       ├── 7
│       │   └── 00000000000000000000
│       ├── 8
│       │   └── 00000000000000000000
│       └── 9
│           └── 00000000000000000000
├── index
│   └── 20200826182922776
└── lock

21 directories, 31 files

```
