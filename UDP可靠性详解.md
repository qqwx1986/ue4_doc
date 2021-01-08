# UDP数据可靠性详解
基于UE4.25.3

NetDriver驱动UNetConnection::FlushNet 最小会在0.2秒
（配置位于UNetDriver::KeepAliveTime）调用一次，如果没有数据包，发个心跳空包过去

## 基础概念
- Bunch 数据包
- FOutBunch 发送数据包
- FInBunch 接收数据包
- FNetPacketNotify 序列号确认管理器

## 包头解析

### 发送
FNetPacketNotify::WriteHeader<br> 
先是序列化四个字节的序列号信息<br>
| 14位OutSeq | 14位InAckSeq | 4位HistrotyWordCount |
- OutSeq 最后即将发送包的序列号
- InAckSeq 最后收到的ACK确认序列号
- HistrotyWordCount 历史序列号Word（一个Word四个字节）数
然后就是序列化HistrotyWordCount个历史序列号信息

### 接收
FNetPacketNotify::ReadHeader<br> 
- 解析出序列号信息到 FNetPacketNotify::FNotificationHeader
- FNetPacketNotify::ProcessReceivedAcks解析出Ack序列号相关信息到FNotificationHeader中，其中包含ACK确认逻辑和丢包的逻辑，确认ACK交由UNetConnection::ReceivedAck回调处理，丢失的包由UNetConnection::ReceivedNak处理
- 因为会存在一个逻辑包拆包，UChannel::ReceivedNextBunch会处理合并Bunch的逻辑

### FNetPacketNotify详解
部分代码

```
AckRecordT AckRecord;			// Track acked seq for each sent packet to track size of ack history
SIZE_T WrittenHistoryWordCount;		// Bookkeeping to track if we can update data
SequenceNumberT WrittenInAckSeq;	// When we call CommitAndIncrementOutSequence this will be committed along with the current outgoing sequence number for bookkeeping

// Track incoming sequence data
SequenceHistoryT InSeqHistory;		// BitBuffer containing a bitfield describing the history of received packets
SequenceNumberT InSeq;			// Last sequence number received and accepted from remote
SequenceNumberT InAckSeq;		// Last sequence number received from remote that we have acknowledged, this is needed since we support accepting a packet but explicitly not acknowledge it as received.
SequenceNumberT InAckSeqAck;		// Last sequence number received from remote that we have acknowledged and also knows that the remote has received the ack, used to calculate how big our history must be

// Track outgoing sequence data
SequenceNumberT OutSeq;			// Outgoing sequence number
SequenceNumberT OutAckSeq;		// Last sequence number that we know that the remote side have received.
```
