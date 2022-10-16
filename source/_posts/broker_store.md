---
title: RocketMQ存储机制详解
date: 2022/10/10
tag: RocketMQ
author: Slide
categories: RocketMQ
---

RocketMQ 实现了灵活的多分区和多副本机制，有效的避免了集群内单点故障对于整体服务可用性的影响。存储机制和高可用策略是 RocketMQ 稳定性的核心，社区上关于 RocketMQ 目前存储实现的分析与讨论一直是一个热议的话题。 本文主要对RocketMQ中三个关键的存储文件CommitLog, ConsumeQueue, IndexFile源码的解析，所有源码的的版本为4.9.2。

<!--more-->

## CommitLog

主要存储文件以及目录包含以下几个

1. **abort** 用来记录整个broker是否为安全关闭的一个状态标识 在运行起来的时候会创建abort文件 安全关闭的情况下会删除 abort文件
2. **checkpoint** 记录 commitlog，consumequeue, index 文件最后刷盘时间戳
3. **commitlog**  存储所有消息的一个文件夹 该文件夹下包含当前broker接受的所有消息 下面会详细介绍commitlog存储消息的格式
4. **config**  broker运行的一些配置文件 以及consumer的消费进度等
5. **consumequeue**  topic对应消费队列的存储 每一个topic下面为每一个queue创建了一个目录目录名称为queue的id 下面会详细描述consumequeue的结构和作用
6. **index** 消息索引 存储的是带有key的Message 下面为详细描述Index文件的作用以及结构
7. **lock** 运行期间用到的全局锁





### 写入方式

在`MessageStore`的接口定义了几种PutMessage的方式

```java
// 异步写入返回CompletableFuture
default CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
    return CompletableFuture.completedFuture(putMessage(msg));
}

// 异步批量写入 返回CompletableFuture
default CompletableFuture<PutMessageResult> asyncPutMessages(final MessageExtBatch messageExtBatch) {
    return CompletableFuture.completedFuture(putMessages(messageExtBatch));
}

// 同步写入直接返回PutMessageResult
PutMessageResult putMessage(final MessageExtBrokerInner msg);

//同步批量写入PutMessageResult
PutMessageResult putMessages(final MessageExtBatch messageExtBatch);

```



### 数据格式

commitlog目录下存储的就是对应具体消息，commitlog文件名称是由20个十进制的数字组成的 表示当前commitlog文件起始的文件偏移量 第一个文件肯定是`00000000000000000000`，每一个commitlog文件大小都是固定的1G，但是对应的内容是**小余等于1G**，消息为一个整体单元不可分割所以是小余等于1G的。

| **属性**                  | **描述**                                                 | 类型       |
| ------------------------- | -------------------------------------------------------- | ---------- |
| msgLen                    | 消息长度                                                 | Int        |
| bodyCrc                   | checksum 校验                                            | Int        |
| queueId                   | 队列id                                                   | Long       |
| flag                      | 用户自定义                                               | Int        |
| queueOffset               | 队列偏移量                                               | Long       |
| physicalOffset            | 磁盘文件偏移量                                           | Long       |
| sysFlag                   | 用来计算FilterType和事务状态                             | Int        |
| bornHostTimestamp         | 消息创建时间                                             | Long       |
| bornHost                  | 生产者Host, ipv4: IP(4)+Port(4) 8 , Ipv6: IP(16)+Port(4) | 8 or 20    |
| storeTimeStamp            | 消息存储时间                                             | Long       |
| storehostAddressLength    | broker地址                                               | 8 or 20    |
| reconsumeTimes            | 回收时间                                                 | Int        |
| preparedTransactionOffset | 预处理事务消息的偏移量                                   | Int        |
| bodyLength                | 消息体长度                                               | Int        |
| body                      | 消息体                                                   | bodyLength |
| topicLength               | 消息归属Topic                                            |            |
| propertiesLength          | 配置信息                                                 |            |



### 写入机制

#### 发送消息请求

```java
// 解析消息头 包含消息生产者发送的一些信息以及topic和队列信息
SendMessageRequestHeader requestHeader = parseRequestHeader(request);
if (requestHeader == null) {
    return CompletableFuture.completedFuture(null);
}
// 消息的上下文 可以用于跟踪消息trace 基本上就是reqeustHeader中的一些信息
mqtraceContext = buildMsgContext(ctx, requestHeader);
//注册的一些消息hook
this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
if (requestHeader.isBatch()) {
    //批量消息
    return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
} else {
    //单条消息
    return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
}

```

```java
private CompletableFuture<RemotingCommand> asyncSendMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                            SendMessageContext mqtraceContext,
                                                            SendMessageRequestHeader requestHeader) {
  
  
 
   int queueIdInt = requestHeader.getQueueId();
   //获取topic的配置信息
   TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    ....
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    ... //省去填充Message数据部分

    CompletableFuture<PutMessageResult> putMessageResult = null;
    String transFlag = origProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
    if (transFlag != null && Boolean.parseBoolean(transFlag)) {
        //事物消息存储
        putMessageResult = this.brokerController.getTransactionalMessageService().asyncPrepareMessage(msgInner);
    } else {
        //普通消息存储
        putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
    }
    return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt);
}

```

`getMessageStore`的实现默认为`DefaultMessageStore` 最后实现消息存储的为`CommitLog`或`DLedgerCommitLog`，我们主要看下`CommitLog`的实现。

#### 处理消息存储

```java
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {

     ... //省略判断以及填充逻辑
     
    //获取消息的编码器
    PutMessageThreadLocal putMessageThreadLocal =  this.putMessageThreadLocal.get();
    //将消息进行编码 对应实现在`MessageExtEncoder`中 主要将上面说的消息结构 编码成bytebuffer
    PutMessageResult encodeResult = putMessageThreadLocal.getEncoder().encode(msg);
    //将编码结果赋值给msg的EncodeBuffer
   msg.setEncodedBuff(putMessageThreadLocal.getEncoder().encoderBuffer);

    //写入锁 这里的锁可以通过配置文件使用 CAS实现的自旋锁 或者ReentrantLock
    putMessageLock.lock(); 
    try {
        //获取对应的MappedFile 最后一个 没有即为空
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
           ....
          
        if (null == mappedFile || mappedFile.isFull()) {
            //当mappedfile为空 或者已经写满了 则新创建一个mappedfile
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); 
        }
         .....
         // 将消息追加到mappedFile中
        result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
        switch (result.getStatus()) {
           //....
            case END_OF_FILE:
                unlockMappedFile = mappedFile;
                // 写入文件过大 超过整个mappedFile的大小 新创建文件再次写入 commitlog文件尾部留有默认长度为 8个字节
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                //....
                result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
                break;
           
        }

        elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        putMessageLock.unlock();
    }


```

通过上面代码我们可以看出来 文件创建的时机有两种，一种是第一次消息存储 内存中没有对应的`mappedFile`文件 则通过`mappedFileQueue.getLastMappedFile(0)`方式创建新的mappedFile文件， 第二种是当前最新的`mappedFile`已经无法写入新的消息了 则创建新的`mappedFile`文件.

#### MappedFile文件创建

```java
if (mappedFileLast != null && mappedFileLast.isFull()) {
    createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
}

if (createOffset != -1 && needCreate) {
    // nextFilePath 就是最后一个文件起始的offset + 最后一个文件的大小
    String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);
    
    String nextNextFilePath = this.storePath + File.separator
        + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
    MappedFile mappedFile = null;
    ..... //省略部分逻辑
       mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,
            nextNextFilePath, this.mappedFileSize)

   
}

```

```java
    /**
     * Only interrupted by the external thread, will return false
     */
    private boolean mmapOperation() {
        boolean isSuccess = false;
        AllocateRequest req = null;
        try {
            // 从队列中获取数据
            req = this.requestQueue.take();
           ...//省略部分过程


            if (req.getMappedFile() == null) {
                long beginTime = System.currentTimeMillis();

                MappedFile mappedFile;
               // 判断是否开启 isTransientStorePoolEnable ，如果开启则使用堆外内存进行写入数据，最后从堆外内存中 commit 到 FileChannel 中。
                if (messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                    try {
                        mappedFile = ServiceLoader.load(MappedFile.class).iterator().next();
                        mappedFile.init(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
                    } catch (RuntimeException e) {
                        log.warn("Use default implementation.");
                        mappedFile = new MappedFile(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
                    }
                } else {
                    //使用mmap(RandomAccessFile#channel#map方法创建mappedByteBuffer)的方式创建MappedFile
                    mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
                }

                long elapsedTime = UtilAll.computeElapsedTimeMilliseconds(beginTime);
                if (elapsedTime > 10) {
                    int queueSize = this.requestQueue.size();
                    log.warn("create mappedFile spent time(ms) " + elapsedTime + " queue size " + queueSize
                        + " " + req.getFilePath() + " " + req.getFileSize());
                }

            // 判断文件大小是否大于等于1G只有commitlog才是1G文件 这里其实就是判断是否为 commitlog文件 数据预热 填充对应pagecache的数据 将mmap的数据内存映射提前准备好
                if (mappedFile.getFileSize() >= this.messageStore.getMessageStoreConfig()
                    .getMappedFileSizeCommitLog()
                    &&
                    this.messageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
                    mappedFile.warmMappedFile(this.messageStore.getMessageStoreConfig().getFlushDiskType(),
                        this.messageStore.getMessageStoreConfig().getFlushLeastPagesWhenWarmMapedFile());
                }

                req.setMappedFile(mappedFile);
                this.hasException = false;
                isSuccess = true;
            }
        } 
        ....
```

这里有几个关键点需要说明下 `isTransientStorePoolEnable`以及 `warmMappedFile`方法

**isTransientStorePoolEnable**
这个必须要在异步刷盘而且为master节点上才能开启, 主要是将数据写入到堆外内存中 然后通过批量commit写入到FileChannel中 这会影响到 复制到Slave节点，复制到Slave节点的数据都是已经commit的数据才会复制，所以可能会导致slave节点的数据有延迟 这个延迟受`commitIntervalCommitLog`,`commitCommitLogThoroughInterval` 默认为`200ms`配置影响



### 文件过期时间

消息是被顺序存储在commitlog文件中的，commitlog中因为每条消息的大小是不固定的 所以消息清理是按照时间单位进行清理的

`DefaultMessageStore`中会创建一个schedule 每10s中检查一次过期文件

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        DefaultMessageStore.this.cleanFilesPeriodically();
    }
}, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);

```

清理文件包含`commitLog`和`consumeQueue`

```java
private void cleanFilesPeriodically() {
    this.cleanCommitLogService.run();
    this.cleanConsumeQueueService.run();
}

```



`commitlog`中判断删除条件如下

```java
private void deleteExpiredFiles() {
    int deleteCount = 0;
    // 配置文件中的过期时间 小时为单位
    long fileReservedTime = DefaultMessageStore.this.getMessageStoreConfig().getFileReservedTime();
    
     ....
     //清理时间达到 默认为凌晨4点 deleteWhen
    boolean timeup = this.isTimeToDelete();
    // 磁盘空间占用率 默认为75%
    boolean spacefull = this.isSpaceToDelete();
    // 手动删除
    boolean manualDelete = this.manualDeleteFileSeveralTimes > 0;
   
    if (timeup || spacefull || manualDelete) {

        if (manualDelete)
            this.manualDeleteFileSeveralTimes--;

        boolean cleanAtOnce = DefaultMessageStore.this.getMessageStoreConfig().isCleanFileForciblyEnable() && this.cleanImmediately;

        log.info("begin to delete before {} hours file. timeup: {} spacefull: {} manualDeleteFileSeveralTimes: {} cleanAtOnce: {}",
            fileReservedTime,
            timeup,
            spacefull,
            manualDeleteFileSeveralTimes,
            cleanAtOnce);

        fileReservedTime *= 60 * 60 * 1000;

        deleteCount = DefaultMessageStore.this.commitLog.deleteExpiredFile(fileReservedTime, deletePhysicFilesInterval,
            destroyMapedFileIntervalForcibly, cleanAtOnce);
        if (deleteCount > 0) {
        } else if (spacefull) {
            log.warn("disk space will be full soon, but delete file failed.");
        }
    }
}

```

默认过期时间为72小时也就是3天，除了我们自动清理，下面几种情况也会自动清理 无论文件是否被消费过都会被清理

1、文件过期并且达到清理时间 默认是凌晨4点，自动清理过期时间
2、文件过期 磁盘空间占用率超过75%后，无论是否到达清理时间 都会自动清理过期时间
3、磁盘占用率达到清理阈值 默认85%后，按照设定好的清理规则(默认是时间最早的)清理文件，无论是否过期
4、磁盘占用率达到90%后，broker拒绝消息写入



## ConsumeQueue

### 作用

Rocketmq是通过订阅Topic来消费消息的，但是因为`commitlog`是不区分topic存储消息的，如果消费者通过遍历commitlog去消费消息 那么效率就非常低下了，所以设计了`ConsumeQueue`用来存储Topic下面每一个队列中消费的offset可以将这里理解为一个队列消息对应的索引文件。



### 数据格式


```
┌────────────────────┬─────────────────┬──────────────────┐
│                    │                 │                  │
│  CommitLog offset  │     Size        │  Tag hashcode    │
│                    │                 │                  │
│        8 byte      │    4 byte       │    8 byte        │
│                    │                 │                  │
└────────────────────┴─────────────────┴──────────────────┘
```

单条ConsumeQueue固定的大小是占用`20`个字节，所以每一个Consume的大小都是固定的 Consume单个文件的大小也是固定的 总共是30万条数据。

```
public static final int CQ_STORE_UNIT_SIZE = 20;

// ConsumeQueue file size,default is 30W
private int mappedFileSizeConsumeQueue = 300000 * ConsumeQueue.CQ_STORE_UNIT_SIZE;

```



### 创建时机

创建`ConsumeQueue`的时机其实是通过查询消息的时候 `consumeQueueTable`中没有当前`topic#queueid`的时候就会创建一个对应的的mappedFile文件

```java
public ConsumeQueue findConsumeQueue(String topic, int queueId) {
   ....
   ConsumeQueue logic = map.get(queueId);
   if (null == logic) {
       //若当前内存中没有对应topic下的consumequeue 则创建对应的consumequeue
       ConsumeQueue newLogic = new ConsumeQueue(
           topic,
           queueId,
           StorePathConfigHelper.getStorePathConsumeQueue(this.messageStoreConfig.getStorePathRootDir()),
           this.getMessageStoreConfig().getMappedFileSizeConsumeQueue(),
           this);
       ConsumeQueue oldLogic = map.putIfAbsent(queueId, newLogic);
       if (oldLogic != null) {
           logic = oldLogic;
       } else {
           logic = newLogic;
       }
   }

   return logic;
}

```

调用`findConsumeQueue`的位置有很多比如通过Topic和queueId获取某个队列消息的时候会使用到

### 数据恢复

在Broker启动的时候会触发`DefaultMessageStore#load`的方法将本地磁盘中的数据 load到内存。

```java
public boolean load() {
        ...
        // load Commit Log
        result = result && this.commitLog.load();

        // load Consume Queue
        result = result && this.loadConsumeQueue();
        ...
}


private long recoverConsumeQueue() {
    long maxPhysicOffset = -1;
    for (ConcurrentMap<Integer, ConsumeQueue> maps : this.consumeQueueTable.values()) {
        for (ConsumeQueue logic : maps.values()) {
            logic.recover();
            if (logic.getMaxPhysicOffset() > maxPhysicOffset) {
                maxPhysicOffset = logic.getMaxPhysicOffset();
            }
        }
    }

    return maxPhysicOffset;
}



```

可以看到在`DefaultMessageStore#load`方法有一个方法`loadConsumeQueue`的方法

```java
private boolean loadConsumeQueue() {
    File dirLogic = new File(StorePathConfigHelper.getStorePathConsumeQueue(this.messageStoreConfig.getStorePathRootDir()));
    //获取Store consumequeue目录下所有子目录 加载进来
    File[] fileTopicList = dirLogic.listFiles();
    if (fileTopicList != null) {
        
        for (File fileTopic : fileTopicList) {
            String topic = fileTopic.getName();

            File[] fileQueueIdList = fileTopic.listFiles();
            if (fileQueueIdList != null) {
                for (File fileQueueId : fileQueueIdList) {
                    int queueId;
                    try {
                        queueId = Integer.parseInt(fileQueueId.getName());
                    } catch (NumberFormatException e) {
                        continue;
                    }
                    ConsumeQueue logic = new ConsumeQueue(
                        topic,
                        queueId,
                        StorePathConfigHelper.getStorePathConsumeQueue(this.messageStoreConfig.getStorePathRootDir()),
                        this.getMessageStoreConfig().getMappedFileSizeConsumeQueue(),
                        this);
                    this.putConsumeQueue(topic, queueId, logic);
                    
                }
            }
        }
    }

    log.info("load logics queue all over, OK");

    return true;
}

```

`load`完之后会通过`recover`将数据初始化到堆外内存中

```java
public void recover() {
    final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
    if (!mappedFiles.isEmpty()) {

        int index = mappedFiles.size() - 3;
        if (index < 0)
            index = 0;

        int mappedFileSizeLogics = this.mappedFileSize;
        MappedFile mappedFile = mappedFiles.get(index);
        ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
        long processOffset = mappedFile.getFileFromOffset();
        long mappedFileOffset = 0;
        long maxExtAddr = 1;
        while (true) {
            for (int i = 0; i < mappedFileSizeLogics; i += CQ_STORE_UNIT_SIZE)          {
                ....//省略文件读取
                //记录最大物理偏移
                this.maxPhysicOffset = offset + size;

                .....
         }
        }
        //记录当前文件的offset
        processOffset += mappedFileOffset;
        this.mappedFileQueue.setFlushedWhere(processOffset);
        this.mappedFileQueue.setCommittedWhere(processOffset);
        this.mappedFileQueue.truncateDirtyFiles(processOffset);

        if (isExtReadEnable()) {
            this.consumeQueueExt.recover();
            log.info("Truncate consume queue extend file by max {}", maxExtAddr);
            this.consumeQueueExt.truncateByMaxAddress(maxExtAddr);
        }
    }
}

```

### 获取消息

```java
public SelectMappedBufferResult getIndexBuffer(final long startIndex) {
   int mappedFileSize = this.mappedFileSize;
   //通过startIndex * 单条消息的大小定位offset
   long offset = startIndex * CQ_STORE_UNIT_SIZE;
   if (offset >= this.getMinLogicOffset()) {
       MappedFile mappedFile = this.mappedFileQueue.findMappedFileByOffset(offset);
       if (mappedFile != null) {
           SelectMappedBufferResult result = mappedFile.selectMappedBuffer((int) (offset % mappedFileSize));
           return result;
       }
   }
   return null;
}

```

以上代码对应`ConsumeQueue#getIndexBuffer`方法，该方法主要通过`startIndex`来获取消费消息的条目，因为每个条目的大小是固定的 所以只需要根据index*20则可以定位到具体的offset的值，就可以知道具体条目的数据`SelectMappedBufferResult`也就是开始消费的条目 然后往后消费，具体代码在`DefaultMessageStore#getMessages`方法中.

```
...
for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
    long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
    int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
    long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
    ....
    
    SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);

    }

```

上面已经说过每一个条目中包含`commitlogOffset，size,tagCode`三个数据，那么`commitLog.getMessage(offsetPy, sizePy)`获取对应位置上的消息.

### 根据时间获取消息

在`ConsumeQueue#getOffsetInQueueByTime`中实现了对消息进行时间偏移读取，但是在`consumeQueue`并没有记录消息的时间，所以避免不了去`commitlog`中获取对应消息的偏移量，我们看下consumeQueue是如何实现的。

```
public long getOffsetInQueueByTime(final long timestamp) {
    MappedFile mappedFile = this.mappedFileQueue.getMappedFileByTime(timestamp);
    if (mappedFile != null) {
        long offset = 0;
        int low = minLogicOffset > mappedFile.getFileFromOffset() ? (int) (minLogicOffset - mappedFile.getFileFromOffset()) : 0;
        int high = 0;
        int midOffset = -1, targetOffset = -1, leftOffset = -1, rightOffset = -1;
        long leftIndexValue = -1L, rightIndexValue = -1L;
        long minPhysicOffset = this.defaultMessageStore.getMinPhyOffset();
        SelectMappedBufferResult sbr = mappedFile.selectMappedBuffer(0);
        if (null != sbr) {
            ByteBuffer byteBuffer = sbr.getByteBuffer();
            high = byteBuffer.limit() - CQ_STORE_UNIT_SIZE;
            try {
                while (high >= low) {
                    //通过最小偏移量和最大偏移量获取中位偏移量
                    midOffset = (low + high) / (2 * CQ_STORE_UNIT_SIZE) * CQ_STORE_UNIT_SIZE;
                    //...省略数据读取
                    //从commitlog中获取当前位置消息的时间
                    long storeTime =
                        this.defaultMessageStore.getCommitLog().pickupStoreTimestamp(phyOffset, size);
                    if (storeTime < 0) {
                        return 0;
                    } else if (storeTime == timestamp) {
                        targetOffset = midOffset;
                        break;
                    } else if (storeTime > timestamp) {
                         // 如果消息大于当前中位消息时间 则再往前进行二分查找
                        high = midOffset - CQ_STORE_UNIT_SIZE;
                        rightOffset = midOffset;
                        rightIndexValue = storeTime;
                    } else {
                       // 如果消息大于当前中位消息时间 则再往后进行二分查找
                        low = midOffset + CQ_STORE_UNIT_SIZE;
                        leftOffset = midOffset;
                        leftIndexValue = storeTime;
                    }
                }

                if (targetOffset != -1) {

                    offset = targetOffset;
                } else {
                    if (leftIndexValue == -1) {

                        offset = rightOffset;
                    } else if (rightIndexValue == -1) {

                        offset = leftOffset;
                    } else {
                        offset =
                            Math.abs(timestamp - leftIndexValue) > Math.abs(timestamp
                                - rightIndexValue) ? rightOffset : leftOffset;
                    }
                }

                return (mappedFile.getFileFromOffset() + offset) / CQ_STORE_UNIT_SIZE;
            } finally {
                sbr.release();
            }
        }
    }
    return 0;
}

```

通过上面我们可以看出 rocketmq通过二分查找`consumeQueue`的方式去`commitlog`中检索对应消息的时间点进行比较。



## IndexFile

### 作用

IndexFile 文件存储的是 包含`Key`的消息，当生产者生产消息到Broker的时候，Broker接收消息的是发现消息包含Key的时候 会将对应消息的索引记录在IndexFile中,只记录包含`Key`的消息是因为 RocketMq 可以通过制定key查询消息

IndexFile 文件名称是已创建文件时间的时间戳命令 `20210901183407523`

### 数据格式

```
┌────────────────────┬────────────────────┬──────────────────────┐
│                    │                    │                      │
│    Header          │       Slots        │        Indexes       │
│                    │                    │                      │
│    40 byte         │    4 byte * 500w   │   20 byte * 2000w    │
│                    │                    │                      │
└────────────────────┴────────────────────┴──────────────────────┘
```

### Header

| 属性              | 描述                      | 类型 |
| ----------------- | ------------------------- | ---- |
| beginTimestamp    | 第一条消息时间戳          | Long |
| endTimestampIndex | 最后一条消息时间戳        | Long |
| beginPhyOffset    | 在commitlog中开始的offset | Long |
| endPhyOffset      | 在commitlog中结束的offset | Long |
| hashSlotCount     | 已使用的SlotCount         | Int  |
| IndexCount        | 索引单元的数量            | Int  |

索引文件中的Header 主要包含以上几个部分 所占用大小就等于 `8 + 8 + 8 + 8 + 4 + 4 = 40`个字节，在RocketMQ源码中我们可以看到`IndexHeader`类中

```java
private static int beginTimestampIndex = 0;
private static int endTimestampIndex = 8;
private static int beginPhyoffsetIndex = 16;
private static int endPhyoffsetIndex = 24;
private static int hashSlotcountIndex = 32;
private static int indexCountIndex = 36;
....

private AtomicLong beginTimestamp = new AtomicLong(0);
private AtomicLong endTimestamp = new AtomicLong(0);
private AtomicLong beginPhyOffset = new AtomicLong(0);
private AtomicLong endPhyOffset = new AtomicLong(0);
private AtomicInteger hashSlotCount = new AtomicInteger(0);
private AtomicInteger indexCount = new AtomicInteger(1);

```

### Slots

Slot是槽位,每一个槽位占用4个字节也就是Int值 表示的是索引单元列表(Indexes)中当前槽位 key最新的索引值(这句话可能有点绕 通过代码和图来理解下)，在`IndexFile`类中的实现

```
public IndexFile(final String fileName, final int hashSlotNum, final int indexNum,
   final long endPhyOffset, final long endTimestamp) throws IOException {

int fileTotalSize =
   IndexHeader.INDEX_HEADER_SIZE + (hashSlotNum * hashSlotSize) + (indexNum * indexSize);
   
 ....
 this.hashSlotNum = hashSlotNum;
}


private int maxHashSlotNum = 5000000;
private int maxIndexNum = 5000000 * 4;
```

fileTotalSize`表示整个IndexFile文件大小是固定的，`maxHashSlotNum`表示最多槽位为500万个,`maxIndexNum`表示最多索引单元为 2000万个 那么可以知道文件大小为`500万*4 + 2000万 * 20 + 40

那么每一个key如何计算所在的槽位呢？ 是通过key的hashcode 取模 500万

```
int keyHash = indexKeyHashMethod(key);
int slotPos = keyHash % this.hashSlotNum;
```

`slotPos`就表示当前消息key所在的槽位索引

每一个槽位中的值 都是**最新**索引单元 所对应`Indexes`中的索引的Position 对应源码中如下

```
// indexHead也就是40B 加上槽位索引*4B 获取当前槽位在文件中的具体position
int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
// 通过position 取得对应槽位上的值
int slotValue = this.mappedByteBuffer.getInt(absSlotPos);
if (slotValue <= invalidIndex || slotValue > this.indexHeader.getIndexCount()) {
    slotValue = invalidIndex;
}

....

// 设置槽位position的最新索引值为当前索引单元的count 也就是插入进去的索引单元在 indexes中的索引
this.mappedByteBuffer.putInt(absSlotPos, this.indexHeader.getIndexCount());


```


### Indexes

index 索引单元存储 数据结构如下

| 属性         | 描述                                    | 类型 |
| ------------ | --------------------------------------- | ---- |
| keyHash      | 消息key的hashCode                       | Int  |
| phyOffset    | 消息在Commitlog文件中的偏移量           | Long |
| timeDiff     | 当前消息和文件起始消息的时间差 精确到秒 | Int  |
| preSlotValue | 上一个索引单元对应的下标值（方便理解）  | Int  |

所以每一个索引单元占用的大小为 `4+8+4+4 = 20B`


### 通过Key查询消息的流程

1. 获取对应Key的HashCode
2. 通过HashCode找到槽位
3. 通过槽位获取最新的Index索引
4. 通过索引中的HasCode比对 依次向上查找 这里还可以通过时间筛选
5. 将对应符合条件的Index单元中的 commitLog的offset记录 查询对应消息