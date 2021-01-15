# UE4登录服务器流程详解
基于4.25.3

之前在 [UDP可靠性详解] 有提过UControlChannel，本文档主要就基于 UControlChannel 的消息<br>
然后再 [UDP握手过程详解] 过程里有完整的握手流程，这里正好接着握手成功后的下面流程处理

## 处理流程
1. Client 端处理Control消息代码在 UPendingNetGame::NotifyControlMessage 
2. Server 端处理Control消息代码在 UWorld::NotifyControlMessage

## 完整流程
1. Server握手完成后，调用 SetExpectedClientLoginMsgType 等待Client发送过来的NMT_Hello消息，设置 UNetConnection::ClientLoginState 为 EClientLoginState::LoggingIn
2. Client确认握手成功后，设置状态State::Initialized，会回调到 UPendingNetGame::SendInitialJoin() 发送 NMT_Hello 消息给Server端
3. Server收到 NMT_Hello消息后<br>
 (1) 如果版本不一致发送 NMT_Upgrade 给Client并关闭连接<br>
 (2) 如果版本验证成功则通过 NMT_Challenge 发送Challenge信息给Client，并等待设置 NMT_Login
4. Client收到 NMT_Challenge 后，解析出Challenge信息本记录在本地（看代码，好像没啥用），然后发送通过 NMT_Login 发送登录信息给Server
5. Server收到 NMT_Login 后，解析出URL相关信息，调用 AGameModeBase::PreLogin 通知处理各个Mode的 PreLogin 事件<br>
 (1) 如果PreLogin事件有处理错误，发送 NMT_Failure 给Client，并不会关闭连接<br>
 (2) Server发送 NMT_Welcome 给Client，此时会带上LevelName和GameName，以及RedirectURL<br>
 (3) 设置 UNetConnection::ClientLoginState 为 EClientLoginState::Welcomed
6. Client收到 NMT_Welcome 后发送 NMT_Netspeed 给 Server，并设置 bSuccessfullyConnected=true
 (1) UEngine::TickWorldTravel 在Check bSuccessfullyConnected==true 时，调用LoadMap加载地图， 如果加载成功成功发送 NMT_Join 给服务器
7. Server收到 NMT_Join <br>
 (1) 调用SpawnPlayActor生成一个新的APlayerController赋值给UNetConnection::PlayerController，初始化APlayerController过程中会调用APlayerController::PostInitializeComponents初始化APlayerState（Server端）这个是同步玩家状态用的，在调用SpawnPlayActor时会一次调用AGameModeBase::Login和AGameModeBase::PostLogin <br>
 (2) 设置 UNetConnection::ClientLoginState 为 EClientLoginState::ReceivedJoin <br>
