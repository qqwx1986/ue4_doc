# UDP可靠性详解
基于版本 4.25.3 分析

NetDriver驱动UNetConnection::FlushNet 最小会在0.2秒（配置位于UNetDriver::KeepAliveTime）调用一次，如果没有数据包，发个心跳空包过去<br>
整个可靠性的逻辑基本分布在两个类中个 UNetConnection和FNetPacketNotify

### Send 发送
FNetPacketNotify::WriteHeader<br> 
先是序列化四个字节的序列号信息<br>
| 14位OutSeq | 14位InAckSeq | 4位HistrotyWordCount |<br>
- OutSeq 最后即将发送包的序列号
- InAckSeq 最后收到的ACK确认序列号
- HistrotyWordCount 历史序列号Word（一个Word四个字节）数<br>
然后就是序列化InSeqHistory历史序列号信息

### Receive 接收
FNetPacketNotify::ReadHeader
- 解析出序列号信息到 FNetPacketNotify::FNotificationHeader
- FNetPacketNotify::ProcessReceivedAcks解析出Ack序列号相关信息到FNotificationHeader中，其中包含ACK确认逻辑和丢包的逻辑，确认ACK交由UNetConnection::ReceivedAck回调处理，丢失的包由UNetConnection::ReceivedNak处理
- 因为会存在一个逻辑包拆包，UChannel::ReceivedNextBunch会处理合并Bunch的逻辑

### FNetPacketNotify详解
为了方便更快的理解，以下举例说明尽量都是以A发送bunch到B的过程 A --> B，

FNotificationHeader中数据信息（为了方便查看对应关系）<br>
FNetPacketNotify::OutSeq ==> FNotificationHeader::Seq<br>
FNetPacketNotify::InAckSeq ==> FNotificationHeader::AckedSeq<br>
FNetPacketNotify::WrittenHistoryWordCount ==> FNotificationHeader::HistoryWordCount<br>
FNetPacketNotify::InSeqHistory ==> FNotificationHeader::History<br>

FNetPacketNotify成员变量（稍作参考，得依赖流程来理解）
1) AckRecord 跟踪记录每个发送packet的序列号信息，会在每个packet发送时push入此刻包的序列号信息以及，然后在收到ack时会pop出来；发送端A使用，实际是为了确认InAckSeqAck用的<br>
我通俗理解如下（有点绕）
- A发送给B包时（此时A会把InAckSeq放到包头也发送给B），假设A此时{OutSeq=100,InAckSeq=2000,InAckSeqAck=1999}（A发送一个序列号100的Packet给B，顺便告诉B, A已经收到B给A发的序列号为2000的Packet了，然后把发送完成顺便把当前记录Push到AckRecord中）
- 当B收到A的PacketId=100的包，此时带过来A的InAckSeq为2000，然后B更新为{InSeq=100，OutAckSeq=2000}（B收到序列号100的Packet，顺便更新下OutAckSeq，此时B知道A收到B发送的序列号2000的Packet了）
- A收到序列号100的Ack，从AckRecord中Pop最后一个Record，比对下是否是100的序列号的Record，如果是，A顺便更新{InAckSeqAck=2000}，即A这边确认B知道A收到2000的Packet了

2) WrittenHistoryWordCount 记录最新一次History新的Word数，一个Word代表可以记录32个包（32bit），就是收到了哪些包的记录，计算方法(PacketNum/32)+1取整；接收端B使用
3) WrittenInAckSeq 在WriteHeader时保存最新的InAckSeq，并在CommitAndIncrementOutSeq时记录到AckRecord中；发送端A使用，实际是为了确认InAckSeqAck用的，配合AckRecord一起使用
4) InSeqHistory 256个bit记录收到包的情况，bit数组，最多记录256个包；接收端B使用
5) InSeq 记录远端的OutSeq，FNetPacketNotify::GetSequenceDelta时会用到；接收端B使用
6) InAckSeq 从远端收到包的最新序列号；通俗来讲就是B现在收到A的包序列号到100了，98和99有可能没收到，反正A最新的100发给B收到了；接收端B使用
7) InAckSeqAck 能确认的远端收到我这边包的最新序列号，会在FNetPacketNotify::GetCurrentSequenceHistoryLength 用到；通俗来讲就是B记录了一个A知道B收到A的包的序列号到98了，其实有可能B已经收到100了，但是A还那边只知道我收到98；接收端B使用
8) OutSeq 发出数据的序列号，在调用CommitAndIncrementOutSeq记录到AckRecord中并自增；通俗来讲就是A发B每一个包都有个序列号，这个就是A发给B最后发包的序列号；发送端A使用
9) OutAckSeq 远端已经收到包的最新序列号，通俗来讲就是A知道B收到的最新序列号（有可能B已经收到100了，但是A这暂时还只知道B收到99）；发送端A使用

几个重要的方法解析
- FNetPacketNotify::GetSequenceDelta 官方注释 获取当前序列与指定头中的序列之间的增量(如果增量为正)；这个是接收端B使用
```cassandraql
SequenceNumberT::DifferenceT GetSequenceDelta(const FNotificationHeader& NotificationData)
{
    /*
        三个判定条件
        1. NotificationData.Seq > InSeq A发过来序列号大于B最后收到的序列号
        2. NotificationData.AckedSeq >= OutAckSeq 这个数据发送反过来 B-->A 数据，我这边理解为 A收到的数据必须大于等于 B能确认A收到的序列号（这个正常情况是一定成立的）
        3. OutSeq > NotificationData.AckedSeq 这个也是 B-->A 数据，我这边理解为 B 发送的数据必须大于 A收到的数据（这个正常情况是一定成立的）
        返回值就是 A发过来的序列号-B最后收到的序列号
    */
    if (NotificationData.Seq > InSeq && NotificationData.AckedSeq >= OutAckSeq && OutSeq > NotificationData.AckedSeq)
    {
        return SequenceNumberT::Diff(NotificationData.Seq, InSeq);
    }
    else
    {
        return 0;
    }
}
```
- TSequenceHistory::AddDeliveryStatus 通俗理解为有个256位的巨型int，每次调用AddDeliveryStatus时整体 左移一位 ，然后最低位填充Delivered状态（0或者1）；接收端B使用
- FNetPacketNotify::GetCurrentSequenceHistoryLength 为InAckSeq-InAckSeqAck的差值，我这边理解是B实际收到100了，A能确定的是B收到98，所以我把99,100的histroy记录（是否收到的记录）数据，SendBunchAck时发送过去；接收端B使用
- TSequenceHistory::IsDelivered 包是否投递成功；发送端A使用，在收到B端SendBunchAck时带的的History数据，判定B是否收到了哪些序列包，哪些包丢失了

先不考虑乱序丢包<br>
为了便于理解一个包的确认发送先拆分为简单的四个过程<br>
SendBunch => RecvBunch => SendBunchAck => RecvBunchAck<br>
因为实际其实只有两个过程 FNetPacketNotify::WriteHeader（SendBunch和SendBunchAck）,FNetPacketNotify::ReadHeader（RecvBunch和RecvBunchAck），如果同时理解这两个过程容易晕，拆成四个过程会容易理解


SendBunch 发送包
1) UNetConnection::FlushNet 在发送一个包时，先调用FNetPacketNotify::WriteHeader压入包头信息，包头信息包括OutSeq（当前PacketId），
2) 会调用FNetPacketNotify::CommitAndIncrementOutSeq，AckRecord压入当前的PacketId信息，然后OutSeq++自增，下次发包时再用

RecvBunch 接收包
1) UNetConnection::ReceivedPacket 接收到一个包
2) FNetPacketNotify::ReadHeader解析包头信息到结构体FNotificationHeader中
3) 通过 FNetPacketNotify::GetSequenceDelta 算出当前收到Packet的Seq与上次记录InSeq的差值PacketSequenceDelta<br>
PacketSequenceDelta==1（说明没丢包没乱序）<br>
PacketSequenceDelta>1 前面的序列号的包可能丢失（如果前面序列包后面收到就是乱序了）<br>
PacketSequenceDelta<0 应该是有乱序包<br>
UNetConnection::InPacketId += PacketSequenceDelta
4) 在调用FNetPacketNotify::Update时,InSeq记录发过来的OutSeq（对方发送过来的PacketId），
5) UNetConnection::AckSeq方法会循环处理UNetConnection::InPacketId 
FNetPacketNotify::InAckSeq 依次递增，最终值等于UNetConnection::InPacketId 
FNetPacketNotify::InSeqHistory 记录收到的包的情况，索引0表示最新的InAckSeq情况，
举个例子
假设调用UNetConnection::AckSeq时，UNetConnection::InPacketId作为参数传进来的值是100，而此时InAckSeq为97，
说明 98,99 号包可能丢失，100收到，此时InSeqHistory记录情况是 [1,0,0,...]

SendBunchAck 发送Ack
ack还是会在FNetPacketNotify::WriteHeader压入包头信息HistrotyWordCount
通过FNetPacketNotify::GetCurrentSequenceHistoryLength()计算收到包的bit数
把InSeqHistory压入包头

RecvBunchAck 接收并解析Ack
FNetPacketNotify::ReadHeader解析InSeqHistory到FNotificationHeader::History中

FNetPacketNotify::ProcessReceivedAcks 中计算FNotificationHeader::AckedSeq与OutAckSeq的差值（即当前收到ack确认包的序列号和上次确认序列号的最大值）AckCount
举个例子，假设发过来的FNotificationHeader::AckedSeq=100 ，当前 OutAckSeq=98，也就是本次ack发送了两个packet包的确认信息（99和100）

调用FNetPacketNotify::UpdateInAckSeqAck 顺便更新InAckSeqAck值
FNetPacketNotify::OutAckSeq最后记录最新的FNotificationHeader::AckedSeq，也就是最新已经确认ack的序列号

```cassandraql
void FNetPacketNotify::ProcessReceivedAcks(const FNotificationHeader& NotificationData, Functor&& InFunc)
{
	if (NotificationData.AckedSeq > OutAckSeq)  //如果是 NotificationData.AckedSeq <= OutAckSeq 可能是重复发了包，直接忽视
	{
		UE_LOG_PACKET_NOTIFY(TEXT("Notification::ProcessReceivedAcks - AckedSeq: %u, OutAckSeq: %u"), NotificationData.AckedSeq.Get(), OutAckSeq.Get());

                // 计算当前包的序列号和上次确认序列号的差值
		SequenceNumberT::DifferenceT AckCount = SequenceNumberT::Diff(NotificationData.AckedSeq, OutAckSeq);

		// Update InAckSeqAck used to track the needed number of bits to transmit our ack history
		InAckSeqAck = UpdateInAckSeqAck(AckCount, NotificationData.AckedSeq);

		// ExpectedAck = OutAckSeq + 1
		SequenceNumberT CurrentAck(OutAckSeq);
		++CurrentAck;
		if (AckCount > (SequenceNumberT::DifferenceT)(SequenceHistoryT::Size))
		{
			UE_LOG_PACKET_NOTIFY_WARNING(TEXT("Notification::ProcessReceivedAcks - Missed Acks: AckedSeq: %u, OutAckSeq: %u, FirstMissingSeq: %u Count: %u"), NotificationData.AckedSeq.Get(), OutAckSeq.Get(), CurrentAck.Get(), AckCount - (SequenceNumberT::DifferenceT)(SequenceHistoryT::Size));
		}

		// Everything not found in the history buffer is treated as lost （如果AckCount大于SequenceHistoryT大小，说明包丢失了）
		while (AckCount > (SequenceNumberT::DifferenceT)(SequenceHistoryT::Size))
		{
			--AckCount;
			InFunc(CurrentAck, false);
			++CurrentAck;
		}

		// For sequence numbers contained in the history we lookup the delivery status from the history
		while (AckCount > 0)
		{
			--AckCount;
			UE_LOG_PACKET_NOTIFY(TEXT("Notification::ProcessReceivedAcks Seq: %u - IsAck: %u HistoryIndex: %u"), CurrentAck.Get(), NotificationData.History.IsDelivered(AckCount) ? 1u : 0u, AckCount);
                        // History 确认对方是否收到包，如果收到会调用UNetConnection::ReceivedAck确认逻辑，如果判定包丢失会调用UNetConnection::ReceivedNak走丢包逻辑重发
			InFunc(CurrentAck, NotificationData.History.IsDelivered(AckCount));
			++CurrentAck;
		}
                //保存当前确认的序列号
		OutAckSeq = NotificationData.AckedSeq;
	}
}

FNetPacketNotify::SequenceNumberT FNetPacketNotify::UpdateInAckSeqAck(SequenceNumberT::DifferenceT AckCount, SequenceNumberT AckedSeq)
{
	if ((SIZE_T)AckCount <= AckRecord.Count())
	{
                // AckRecord记录了上次发送包的序列号信息
                // 大于1 ，直接pop出 AckCount-1的 Record对象
		if (AckCount > 1)       
		{
			AckRecord.PopNoCheck(AckCount - 1); 
		}
                // 取出最新的AckRecord记录
		FSentAckData AckData = AckRecord.PeekNoCheck();
		AckRecord.PopNoCheck();

		// verify that we have a matching sequence number（如果最后发送包的序列号和收到ACK包的序列号一致，那么InAckSeqAck就更新赋值为最新的InAckSeq）
		if (AckData.OutSeq == AckedSeq)
		{
			return AckData.InAckSeq;
		}
	}
	// Pessimistic view, should never occur but we do want to know about it if it would（不会发生）
	ensureMsgf(false, TEXT("FNetPacketNotify::UpdateInAckSeqAck - Failed to find matching AckRecord for %u"), AckedSeq.Get());
	return SequenceNumberT(AckedSeq.Get() - MaxSequenceHistoryLength);
}
```
### 什么时候发包？
发包在 UNetConnection::FlushNet真正的发送到网络
满足三个条件其中之一为true时就会发送一个包
1) SendBuffer.GetNumBits() 这个容易理解，有数据包时发送，通过UNetConnection::SendRawBunch压入数据到SendBuffer中，会在同一帧的FlushNet之前调用
2) HasDirtyAcks HasDirtyAcks会在UNetConnection::ReceivedPacket收到数据包时自增1（非0就是true），因为UNetConnection::FlushNet实在同一个个UWorld::Tick中调用，而且FlushNet在ReceivedPacket后调用
3) Driver->GetElapsedTime() - LastSendTime > Driver->KeepAliveTime && !IsInternalAck() && State != USOCK_Closed 举例上次发送周期达到KeepAliveTime（默认0.2s）时，可以理解为一个心跳包，IsInternalAck()好像是内部测试用的

### 先前介绍下通道UChannel
```cassandraql
class FInBunch*		InRec;				// Incoming data with queued dependencies.
class FOutBunch*	OutRec;				// Outgoing reliable unacked data.
class FInBunch*		InPartialBunch;		// Partial bunch we are receiving (incoming partial bunches are appended to this)
```
InRec 接收端Bunch缓冲区，假设当前处理序列号是100，但是当前发来的，类型tcp的窗口大小
OutRec 发送Bunch缓冲区，类型tcp的窗口大小
InPartialBunch 接收分包Bunch缓冲区

一个UNetConnection有个UChannel的数组，其中有两个Channel是固定的一个 UVoiceChannel 和 UControlChannel，然后其他剩下都是UActorChannel，我这样理解每个Actor都拥有自己的Channel，每个Actor处理自己的相关的网络包逻辑，不会混在一起<br>
所有的Channel都存在 UNetConnection::OpenChannels中，但是一旦有网络活动时才会才会把活跃的Channel放到 UNetConnection::ChannelsToTick 中，一旦不活跃就会从ChannelsToTick删除，活跃就是Bunch::bOpen，不活跃就是Bunch::bClose

### 发包、收包，丢包、乱序，重包？
这些都通过UChannel来处理
1) 发包 UChannel::SendBunch 发送包时如果包bit长度大于MAX_SINGLE_BUNCH_SIZE_BITS时会拆包处理，待发送的FOutBunch放到OutRec队列中，再通过UChannel::SendRawBunch发送最小Bunch包
1) 收包 接收端通过 UChannel::ReceivedRawBunch ，如果收到的包正好安装序列号顺序来的，那么直接走处理包流程，如果不是，那么放到InRec队列中
2) 处理包 没来一个FInBunch会通过 UChannel::ReceivedNextBunch 处理，
- 如果不是一个分包 UChannel::ReceivedSequencedBunch
3) 丢包 如果发现丢包，通过UNetConnection::ReceivedNak处理
4) 乱序 无序的数据会仍在InRec队列中等待处理 UChannel::ReceivedNextBunch 会处理合并包
5) 重包 如果InRec队列中已存在该包，那么丢弃，如果发过来的包ChSequence<=当前已经处理的InReliable的序列号，那么丢弃
```cassandraql
// DataChannel.cpp
if( Bunch.ChSequence==(*InPtr)->ChSequence )
{
    // Already queued.
    return;
}

//NetConnection.cpp
// Ignore if reliable packet has already been processed.
if ( Bunch.bReliable && Bunch.ChSequence <= InReliable[Bunch.ChIndex] )
```

### 怎么拆包和组合？
1) 拆包，上面有提过超过 MAX_SINGLE_BUNCH_SIZE_BITS 就会拆，然后放到OutRec等待发送就行
2) 合并包，接收端发现如果Bunch.bPartial==1时，说明是个分包，如果bPartialInitial==1表示是第一个分包，如果bPartialFinal==1表示是最后一个分包<br>
思考和疑问<br>
假设一个包被拆成3个Bunch，如果接收端收到第一个和第三个，第二个没收到怎么处理，或者到的顺序是 132 || 321 || 312 || 213 || 231 ，好像会丢弃所有的已经收到的分包，需要重传？<br>
如果在同一个Channel里面有同时发了两个大包，都需要拆包，因为我看InPartialBunch只会处理同一个拆包，如果是错开收到了两个不同的拆包子包，好像会丢弃原来的包，需要重传？<br>

### 命令测试 FPacketSimulationSettings
1) Net pktLag=，模拟延迟，单位是毫秒
2) Net PktLagVariance=300，在模拟延迟的基础上，再上下浮动300毫秒。加上这个就会出现移动瞬移卡顿的效果
3) Net PKtLoss=，丢包，单位是百分比，Net PKtLoss=90就是90%会丢包，也会出现移动瞬移卡顿
4) Net PktOrder=1，乱序发包，会出现一定的移动瞬移，但不太明显
5) Net PktDup=，重复发包，单位是百分比，Net PktDup=20表示20%会出现重复发包。
