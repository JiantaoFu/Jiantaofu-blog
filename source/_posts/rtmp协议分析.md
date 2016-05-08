title: rtmp协议分析
comments: true
date: 2016-05-06 22:34:27
tags:
keywords:
description:
thumbnailImage:
coverImage:
photos:
---

#rtmp协议分析

官方的文档在这[下载](https://wwwimages2.adobe.com/content/dam/Adobe/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)

##基本概念
- Message Stream: 用于消息传输的逻辑通道，多个Stream可以复用同一个物理连接。
- Message Stream ID: 用于标识不同的流。
- Chunk: 消息传输需要将其拆分成小的片段，以便复用减少时延。
- Chunk Stream: 用于传输Chunk的逻辑通道，一个Chunk Stream可以复用多个Message Stream。一个物理连接上可以复用多个Chunk Stream。
- Chunk Stream ID: 标识Chunk流

##RTMP chunk stream
这个是设计的基于TCP的传输协议, RTMP协议构建在这个之上。

###Handshake

握手第一阶段是交换版本号看是否能支持，只有一个字节(C0, S0)，可以提高效率，在协议支持的情况下提前断开连接。后面的(C1, S1)和(C2, S2)类似ping的来回，里面带上了timestamp，而且包的大小非常大，用random数据填充，协议没有说明这样做的目的，但提到了可以初步估算RTT和带宽，同时也替代可能不准。

###Chunk

Adobe在Chunk设计上还是有些考量的，花费了许多心思来避免传输重复的字段来做到Header compresssion，也导致协议的设计略微复杂。而且这种做法需要必须做到可靠传输，也就是rtmp需要用tcp的原因。

- 多路数据同一连接multiplexing，小的chunk size可以避免大的Video数据包对Audio数据包的影响。
- 但是，小的chunk会带来更多的CPU开销，并影响传输效率。

Chunk头的设计也是一个长度可变的，包括基本头，消息头，和扩展时间戳字段。

```
 +--------------+----------------+--------------------+--------------+
 | Basic Header | Message Header | Extended Timestamp | Chunk Data   |
 +--------------+----------------+--------------------+--------------+
 |                                                    |
 |<------------------- Chunk Header ----------------->|
```

Basic Header有两个信息，一个是StreamID，还有一个是ChunkType。Basic Header设计为可以变长的(1 Byte, 2 Byte, 3 Byte)，ChunkType是固定的2 bits，变化的是StreamID，将 0, 1预留出来，标识不同长度的StreamID（对应14 bits, 22 bits)， StreamID 2作为内部控制协议预留，Basice Header长度也是一个字节。

```
  0 1 2 3 4 5 6 7
 +-+-+-+-+-+-+-+-+
 |fmt| cs id     |
 +-+-+-+-+-+-+-+-+
 Chunk basic header 1
 
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |fmt| 0         | cs id - 64    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Chunk basic header 2
 
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |fmt| 1         |         cs id - 64            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Chunk basic header 3
```

cs id(StreamID): used to say the type of the message. 2 (low level), 3 (high level), 4 (control stream), 5 (video) and 6 (audio).

Message Header也是可变的，格式由basic header的fmt来标识:

- fmt 0: Chunk Stream刚开始的时候，或者Timestamp回退的时间使用。
- fmt 1: 和fmt 0相比少了StreamID，因为之前的消息有携带过StreamID，这里就略掉了。
- fmt 2: 如果消息长度也都一样，那这个字段也可以省略掉了。
- fmt 3: 长度信息和时间间隔信息也都一样，就都不用传了。

```
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | timestamp                                     |message length |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | message length (cont)       |message type id  | msg stream id |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | message stream id (cont)                      |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Chunk Message Header - Type 0
 
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | timestamp delta                               |message length |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | message length (cont)         |message type id|
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Chunk Message Header - Type 1
  
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | timestamp delta                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Chunk Message Header - Type 2
```

Extended Timestamp只是在3 bytes的timestamp(delta)字段不够用的时候才有的，这样就构成了4 bytes来表示时间戳。

###Message Type

- Set Chunk Size(message type id 1): 设置Chunk大小，默认为128 bytes。
- Abort Message(message type id 2): 链接终止前通知对方，对方可以丢弃未完整收到的chunk数据。
- Ack(message type id 3): 当收到window size（发送方可以最多发送的字节数，而不需要等收到Ack）大小的数据(自上次Ack或者从传输开始）后给对方的通知。
- Ack Window Size(message type id 5): 通知接受方当前发送方使用的window size的大小，用来调整Ack的频率。
- Set Peer Bandwidth(message type id 6): 通知发送方window size的大小。

为什么没有4？4预留给了RTMP的控制协议，后面可以看到RTMP协议里面还定义了其他的一些message Type，上面的这些message type是和传输相关的，而后面定义的那些则是和应用相关。

###参考源码

librtmp包解析函数, [int RTMP_ReadPacket(RTMP *r, RTMPPacket *packet)](https://github.com/lu-zero/rtmpdump/blob/master/librtmp/rtmp.c#L2886)

librtmp对不同message type的处理, [int RTMP_ClientPacket(RTMP *r, RTMPPacket *packet)](https://github.com/lu-zero/rtmpdump/blob/master/librtmp/rtmp.c#L1101)

##RTMP

Adode的文档在概念上不太清晰，需要说明的是RTMP协议是在RTMP Chunk(传输协议)之上的一层应用协议。另外RTMP混杂了太多的功能，而且很多是和Adobe的服务器业务逻辑耦合的。

###公共消息头

```
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Message Type |                  Payload length                |
 | (1 byte)     |                  (3 bytes)                     |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                        Timestamp                              |
 |                        (4 bytes)                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                        Stream ID              |
 |                        (3 bytes)              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Message Header
```

###用户控制消息

控制协议的Chunk Message使用如下的一些字段值:

- Message Stream ID 0
- Chunk Stream ID 2
- Message Type Id 4

```
 +------------------------------+-------------------------
 | Event Type (16 bits)         | Event Data
 +------------------------------+-------------------------
     Payload for the ‘User Control’ protocol message
```

Event包括:

- 0: Stream Begin
- 1: Stream EOF
- 2: Stream Dry
- 3: Set Buffer Length
- 4: Stream Is Recored
- 6: Ping Request
- 7: Ping Response


###Command消息

AMF编码的命令，包括connect, createStream, publish, play, pause, onstatus, result等。

- Message Type Id 20 (AMF0)
- Message Type Id 17 (AMF3)

Command消息包括:

- NetConnection::connect
- NetConnection::call
- NetConnection::CreateStream
- NetStream::play
- NetStream::play2
- NetStream::deleteStream
- NetStream::closeStream
- NetStream::receiveAudio
- NetStream::receiveVideo
- NetStream::publish
- NetStream::seek
- NetStream::pause
- NetStream::onStatus

###Data消息

数据的元信息，如创建时间，持续时间等。

- Message Type Id 18 (AMF0)
- Message Type Id 15 (AMF3)

###Shared Object消息

用于Flash对象的同步（远程调用操作）。

- Message Type Id 19 (AMF0)
- Message Type Id 16 (AMF3)

###Audio消息

传输audio数据

- Message Type Id 8

###Video消息

传输video数据

- Message Type Id 9

###Aggregate消息

聚合其他的消息。

##参考
- [Problems in RTMP](http://blog.kundansingh.com/2009/06/problems-in-rtmp.html)
- [AMF](https://en.wikipedia.org/wiki/Action_Message_Format)
