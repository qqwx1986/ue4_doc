# UE4 UDP握手详解
基于4.25，整个握手通过StatelessConnectHandlerComponent完成（server&client端）

## 包头解析
握手包头总共227个bit+1个bit终止符（29个字节）；
其中第一个bit是bHandshakePacket；
第二个bit是bRestartHandshake；
第三个bit是SecretId；
从第四个bit开始是八个字节的timestamp；
然后是20个自己的cookie信息；
最后填充一个bit的终止符。

## 四个握手包
- client->server NotifyHandshakeBegin bHandshakePacket为1，其他都为0
- server->client SendConnectChallenge bHandshakePacket为1，bRestartHandshake为0，timestamp记录当前的流逝时间戳，cookie为clientAddr&SecretId&Timestamp的加密串，可以反向解密
- client->server SendChallengeResponse 把SendConnectChallenge原包返回去
- server->client SendChallengeAck bHandshakePacket为1，bRestartHandshake为0，timestamp为-1，cookie为SendConnectChallenge中的cookie

## 握手过程
1. Client通过Browse与server建立UDP连接，连接成功后状态为 State::UnInitialized
2. Client在StatelessConnectHandlerComponent::Tick轮训时发现状态State::UnInitialized时调用NotifyHandshakeBegin发送握手消息（此消息间隔1s发送一个，防止丢包）
3. Server端在收到NotifyHandshakeBegin时会根据包的来源地址去连接表中（UNetDriver::MappedClientConnections）查找相关地址的连接，如果没查到，说明是握手包，这是会调用SendConnectChallenge
4. Client在收到SendConnectChallenge时发送SendChallengeResponse，成功后会把状态置为State::InitializedOnLocal，此状态在StatelessConnectHandlerComponent::Tick时会间隔1秒发送一个给Server端，避免丢包
5. Server端在收到SendChallengeResponse后，整个Server确认该连接握手成功，发送SendChallengeAck，并把该连接加入到UNetDriver::MappedClientConnections.Add中
6. client收到SendChallengeAck收状态置为State::Initialized，客户握手成功
