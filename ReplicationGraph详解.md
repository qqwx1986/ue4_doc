# ReplicationGraph 流程分析
基于 4.25.3
## 基础类
1. UReplicationDriver 驱动，这个是个接口类，Graph需要继承于该类
2. UReplicationGraph 管理Actor的同步，一个World只有一个，自己实现一般是继承于该类
3. UReplicationGraphNode 我这边理解Driver的（同步策略）Node，可以自定义，Driver中存了全局的Node（针对所有的连接有效），然后Connect有自己特有的Node
4. UReplicationConnectionDriver，这是接口 连接的 RepGraph 驱动，每个Connect有一个自己的Driver
5. UNetReplicationGraphConnection 继承于UReplicationConnectionDriver，项目可以继承这个再实现自己的Driver

## Driver初始化流程
#### 1. UNetDriver::InitBase 时会New一个UReplicationDriver，这个配置在 DefaultEngine.ini 里面的配置选项 [/Script/OnlineSubsystemUtils.IpNetDriver] ReplicationDriverClassName
#### 2. UNetDriver::SetReplicationDriver 
- 如果老的Driver存在，那么会调用TearDown卸载原来的Driver，然后存在的Conection会调用 UNetConnection::TearDownReplicationConnectionDriver卸载连接的ReplicationConnectDriver 
- 调用 UReplicationGraph::SetRepDriverWorld 设置当前的World 
- 调用 UReplicationGraph::InitForNetDriver 初始化
  1. UReplicationGraph::InitGlobalActorClassSettings 设置Actor相关的Rep信息 <br>
      设置Actor的多播开关 UReplicationGraph::RPC_Multicast_OpenChannelForClass，<br>
      设置全局各个Actor类的FClassReplicationInfo信息，在UReplicationGraph::GlobalActorReplicationInfoMap，<br>
      设置Actor Rep的最大举例 UReplicationGraph::DestructInfoMaxDistanceSquared 
  2. UReplicationGraph::InitGlobalGraphNodes 预设值RepList分配池子，加入自定义的GraphNode 
  3. 循环 UNetDriver的Connect 加入到GraphDriver中 UReplicationGraph::AddClientConnection 
- UReplicationGraph::InitializeActorsInWorld
  1. UReplicationGraph 中所有的Node调用 UReplicationGraphNode::NotifyResetAllNetworkActors
  2. UReplicationGraph 中所有的连接调用 UNetReplicationGraphConnection::NotifyResetAllNetworkActors 
  3. 遍历World中全局所有的Actor，如果ULevel::IsNetActor，调用UReplicationGraph::AddNetworkActor 
     判定Actor::bReplicates 如果为真加入 UReplicationGraph::ActiveNetworkActors 记录了Graph中有哪些Actor在参与，纯粹一个记录 
     加入 GUReplicationGraph::GlobalActorReplicationInfoMap 这个在上面有提过
     UReplicationGraph::RouteAddNetworkActorToNodes 遍历 GlobalGraphNodes ，调用 NotifyAddNetworkActor 通知有新的Actor加入
#### 3. 每当有个新的连接握手完成调用 UNetDriver::AddClientConnection 
- UReplicationGraph::AddClientConnection 加入Connect 到 UReplicationGraph::Connections 中
- 在调用 UReplicationGraph::CreateClientConnectionManagerInternal 创建一个新的 UNetReplicationGraphConnection 时
  1. 会依次调用 UNetReplicationGraphConnection::InitForGraph
  2. UNetReplicationGraphConnection::InitForConnection
  3. UReplicationGraph::InitConnectionGraphNodes，这里可以加入自定义的Node到 UNetReplicationGraphConnection::ConnectionGraphNodes 中，并注册ClientLevelVisible相关事件委托
- UNetDriver::TickFlush 时会调用到 UReplicationGraph::ServerReplicateActors ，可以部分参考[官方文档说明](https://docs.unrealengine.com/en-US/InteractiveExperiences/Networking/Actors/ReplicationFlow/index.html)，里面有部分有理解价值，但是里面流程大部分还是之前非ReplicationGraph的同步方式
- 遍历 UReplicationGraph::PrepareForReplicationNodes 调用 UReplicationGraphNode::PrepareForReplication 
- 遍历 UReplicationGraph::Connections 的每个Connect 
- 首先会发送movement的消息，这个是不想受到流控影响？
- 遍历全局GlobalGraphNodes的 GatherActorListsForConnection 到 FConnectionGatherActorListParameters中 
- 遍历 Connect 自己的 GatherActorListsForConnection 
- UReplicationGraph::ReplicateActorListsForConnections_Default 发送Actors的相关同步信息 
  1. 遍历 Gather 的 Actors Default 列表
  2. 从 GlobalActorReplicationInfoMap 获取Actor的Rep信息，ReadyForNextReplication 判断这一帧是否需要同步
  3. 根据上面的 GlobalActorRep 配置信息算出优先级（TODO: 具体怎么算的，后面有时间再看补上文档） AccumulatedPriority 放到 UReplicationGraph::PrioritizedReplicationList 中，然后根据优先级排序 <br>
  4. 遍历 PrioritizedReplicationList，调用 UReplicationGraph::ReplicateSingleActor
  5. 调用 AActor::CallPreReplication，会遍历该Actor下的所有 ReplicatedComponents 并调用 PreReplication
  6. 调用 UActorChannel::ReplicateActor 
  7. 调用 UActorChannel::ActorReplicator 的 ReplicateProperties(TODO:这里面涉及到FReplicationFlags，后面有时间再看补上文档) 把需要该Actor的同步的数据写入Bunch 
  8. 调用 AActor::ReplicateSubobjects 写入Actor的子对象 ReplicatedComponents，遍历子对象 ReplicatedComponents并分别调用子对象的 ReplicateSubobjects（TODO:怎么序列号到Bunch中的这一块也能写一大篇了）
  9. 调用 UChannel::SendBunch —> UChannel::SendRawBunch -> UNetConnection::SendRawBunch 这个在[可靠性详解]中有说Bunch的发送过程 
- UReplicationGraph::ReplicateActorListsForConnections_FastShared 这个是快速发送（少了个CallPreReplication所以快？）, 目前就看到 UReplicationGraphNode_ActorListFrequencyBuckets 使用了
  1. 遍历 Gather 的 Actors FastShared 列表
  2. 调用 UReplicationGraph::ReplicateSingleActor_FastShared
  3. Bunch 用的是 FFastSharedReplicationInfo::Bunch 引用
  4. 调用 FClassReplicationInfo::FastSharedReplicationFunc（但是没看到这个的默认Func实现） 序列号数据到 FFastSharedReplicationInfo::Bunch 中
  5. SendBunch Merge是0，所以快？
  6. 感觉这个好像这个版本没用
  
## 辅助类
#### 1. FClassReplicationInfo Actor类的Rep配置信息，距离，饥饿（距离上次rep的帧数）等
#### 2. FNetViewer  
```
/** 保存当前的Connection **/
UNetConnection* Connection;
/** The "controlling net object" associated with this view (typically player controller) */
/** 代码逻辑是 如果 UNetConnection::PlayerController不为空那么就是PlayerController，如果为空就是 UNetConnection::OwningActor **/
class AActor* InViewer;
/** The actor that is being directly viewed, usually a pawn.  Could also be the net actor of consequence */
/** 就是 UNetConnection::ViewTarget  **/
class AActor* ViewTarget;
/** Where the viewer is looking from */
FVector ViewLocation;
/** Direction the viewer is looking */
FVector ViewDir;
```
#### 3. FConnectionGatherActorListParameters GatherActorListsForConnection函数的参数
```
// 虽然是个FNetViewer数组，但是每次tick循环其实就一个（目前我看到的）
FNetViewerArray Viewers;
// 当前Gather的ConnectionDriver
UNetReplicationGraphConnection& ConnectionManager;
// 当前的Rep 帧数
uint32 ReplicationFrameNum;
// 需要收集的Actor放到这里面，每个Node手机的Actor列表都独立存放
FGatheredReplicationActorLists& OutGatheredReplicationLists;

```
#### 4. FGlobalActorReplicationInfo World里面全局的Actor的Rep信息，以下列举几个 <br>
- WorldLocation 缓存该Actor的location信息 <br>
- 当前Actor的FClassReplicationInfo的指针 <br>
- 依赖 DependentActorList 列表，当Rep当前Actor时，随后会立刻Rep他的DependentActorList<br>
- ParentActorList，和DependentActorList对应，A中的DependentActorList存了B，那么B中的ParentActorList的存了A，（思考，如果出现环怎么处理？）

#### 5. FGatheredReplicationActorLists <br>
- 包含两个列表，一个是Default一个是FastShared,对应上面的ReplicateActorListsForConnections_Default和ReplicateActorListsForConnections_FastShared <br>
- 里面本质是个 TArray<FActorRepList> ，数组套数组，通俗解释就是每个Node分别收集的ActorRep列表 <br>

#### 6. UNetConnection 好像和Rep没有关系，但是牵扯到上面的 FNetViewer 初始化，为了便于理解 <br>
- 继承于 UPlayer ，UPlayer 中有个 PlayerController，我理解就是控制当前的Connection的 Cotroller <br>
- OwningActor 拥有者，一般和上面UPlayer::PlayerController指向同一个值（遗留，没搞清楚不一般情况指向谁呢？）<br>
- ViewTarget 看注释的意思我理解说的是 OwningActor 控制的 Actor，一般情况也是 UPlayer::PlayerController 同一个值（自己控制自己嘛），如果观战，我的理解这个值就是被观战者的OwingActor？<br>
  
  
## 官方自带的Node（策略）
#### 1. UReplicationGraphNode_ActorList 继承于 UReplicationGraphNode
- 部分变量说明
  1. ReplicationActorList 保存了一组需要Rep 的 Actor 列表
  2. StreamingLevelCollection 类型为  TArray<FStreamingLevelActors>，我看实现本质是个 TArray<FStreamingLevelActors>，其中FStreamingLevelActors，记录了一个StreamingLevelName，（挖个坑，这个暂时没看明白干嘛的）
- Gather干了啥
  1. 整个 ReplicationActorList（本质是个 Actor* 的数组）
  2. StreamingLevelCollection 判断对于Connection的可见/相关性，可见的才会Gather，只会Gather(相关/可见)性Level中的Actors
  3. 基类中的 UReplicationGraphNode::AllChildNodes，递归Gather功能，基类中虽然有 AllChildNodes，但是实现了 TearDown等清理功能，没有实现Gather功能
- 使用场景，作为基类，好像没直接使用

#### 2. UReplicationGraphNode_ActorListFrequencyBuckets 继承于 UReplicationGraphNode
- 部分变量说明
  1. NonStreamingCollection NonStreamingCollection列表，（挖个坑，Steaming和NonStreaming有啥区别，我看到代码中取得是ULevel的名字，如果取得是空那么是NonStreaming）
  2. StreamingLevelCollection StreamingLevel列表
- Gather干了啥
  1. StreamingLevelCollection 判断对于Connection的可见/相关性，可见的才会Gather，只会Gather(相关/可见)性Level中的Actors
  2. 当前帧数的取模 NonStreamingCollection[frame/num]
- 使用场景
  1. UReplicationGraphNode_GridCell 中在用
  
#### 3. UReplicationGraphNode_DynamicSpatialFrequency 继承于 UReplicationGraphNode_ActorList
- Gather干了啥
  1. 没有实现自己的Gather，用的基类的Gather
- 使用场景
  1. 暂时没看到使用场景
  
#### 4. UReplicationGraphNode_ConnectionDormancyNode 继承于 UReplicationGraphNode_ActorList
- Gather干了啥
  1. 从 ReplicationActorList 删除 bDormantOnConnection 的 Actor，ReplicationActorList 剩下的Add进入 OutGatheredReplicationLists 中
  2. 从 StreamingLevelCollection 删除 bDormantOnConnection 的SteamingLevelActor ， StreamingLevelCollection 剩下的 Add 进 OutGatheredReplicationLists 中，其中删除的会缓存到RemovedStreamingLevelActorListCollection 
- 使用场景
  1. UReplicationGraphNode_DormancyNode 存了 UNetReplicationGraphConnection 到 UReplicationGraphNode_ConnectionDormancyNode 的映射
  
#### 5. UReplicationGraphNode_DormancyNode 继承于 UReplicationGraphNode_ActorList
- Gather干了啥
  1. 获取当前Connection的UReplicationGraphNode_ConnectionDormancyNode，然后Gather
- 使用场景
  1. UReplicationGraphNode_GridCell 中使用
  
#### 6. UReplicationGraphNode_GridCell 继承于 UReplicationGraphNode_ActorList
- 部分变量说明
  1. CreateDynamicNodeOverride 这是个创建DynamicNode的function，可以自己定制
  2. DynamicNode 动态Node，如果CreateDynamicNodeOverride不为空，那么CreateDynamicNodeOverride创建，否则就是个 UReplicationGraphNode_ActorListFrequencyBuckets
  3. DormancyNode 类型为 UReplicationGraphNode_DormancyNode
- Gather干了啥
  1. 没有实现自己的Gather，用的基类的Gather，所以这里的Gather没有用到上面的 DynamicNode 和 DormancyNode
- 使用场景
  1. UReplicationGraphNode_GridSpatialization2D使用，类似九宫格的格子
  
#### 7. UReplicationGraphNode_GridSpatialization2D 继承于 UReplicationGraphNode_ActorList
- 部分变量说明
  1. CellSize 格子大小
  2. SpatialBias 空间偏移
  3. bDestroyDormantDynamicActors  (When enabled the RepGraph tells clients to destroy dormant dynamic actors when they go out of relevancy)
  4. CreateCellNodeOverride 创建 UReplicationGraphNode_GridCell 的function, 就是可以自定义 GridCell，如果为空那就是用默认的
  5. DynamicSpatializedActors 动态Actors
  6. StaticSpatializedActors pending静态Actors
  7. PendingStaticSpatializedActors 静态Actors
  8. Grid 二维格子数组
  9. GatheredNodes Gather用的缓存TArray，可能为了效率吧
- PrepareForReplication 干了啥（这个会在Gather之前调用）
  1. 遍历 DynamicSpatializedActors
  2. 获取 Actor 当前的 NewCellInfo
  3. 比对计算 Actor 的 PreCellInfo 和 NewCellInfo（其实就是看动了没，如果动了是不是跨Grid了），<br>
     这里有个地方需要特别理解，和传统有点思路有点不一样，这边会调用 UReplicationGraphNode_GridSpatialization2D::GetCellInfoForActor 根据配置参数（FClassReplicationInfo::CullDistance 视距？ CellSize）算出次Actor的相关Grids，我理解就是我可能被别人看到的几个相关Grid<br>
     然后计算 OldCellGrids 调用RemoveDynamicActor，遍历 NewCellGrids 调用 AddDynamicActor加入，我这边理解就是同一个Actor可能会放在不同的CellGrid中 <br>
     PreCellInfo 更新为 NewCellInfo <br>
  4. 遍历 PendingStaticSpatializedActors 中的 Actor 放入到 StaticSpatializedActors 中，并从PendingStaticSpatializedActors删除
- Gather干了啥
  1. 获取当前Connection的 CellGrid ,Gather 当前的 CellGrid，上面 Prepare 已经说明了和这个CellGrid的Actor已经放进来了（及时坐标不在这个CellGrid内）
  2. 如果 bDestroyDormantDynamicActors 为 true，那么 Connection的 PreCell 的 Actor 放到 PrevDormantActorList 列表中<br>
     遍历 PrevDormantActorList，加入到当前GraphConnection 的 PendingDormantDestructList 中，<br>
     这个会在调用 UReplicationGraph::ServerReplicateActors 时会调用  UNetReplicationGraphConnection::ReplicateDormantDestructionInfos 同步给 ConnectClient
- 使用场景
  1. 基本都可以用上，可以自己定制
  
#### 8. UReplicationGraphNode_AlwaysRelevant 继承于 UReplicationGraphNode
- 部分变量说明
  1. ChildNode 类型是UReplicationGraphNode_ActorList
  2. AlwaysRelevantClasses AlwaysRelevant Class列表，显示调用AddAlwaysRelevantClass插入
- Gather干了啥
  1. Gather ChildNode
- 使用场景
  1. 暂时没看到使用场景
  
#### 9. UReplicationGraphNode_AlwaysRelevant_ForConnection 继承于 UReplicationGraphNode_ActorList
- 部分变量说明
  1. ReplicationActorList
  2. PastRelevantActors
- Gather干了啥
  1. 调用基类 UReplicationGraphNode_ActorList 的Gather
  2. ReplicationActorList 重置 Reset 为空
  3. 最终我看就是Conection的 PlayerController 和 ViewTarget
- 使用场景
  1. 暂时没看到使用场景，类似倒计时、毒圈可以用这个做吧
  
#### 10. UReplicationGraphNode_TearOff_ForConnection 继承于 UReplicationGraphNode
- 部分变量说明
  1. TearOffActors 记录了所有 TearOff 状态的 Actor，当加入该队列时，会通知相应的Channel下一次Rep时关闭
  2. ReplicationActorList
- Gather干了啥
  1. 如果TearOffActors.Num()==0 啥也不干 
  2. 倒序遍历（倒序是为了下面删除的时候效率高，因为是个数组）TearOffActors，保证置成TearOff状态的Actor会Rep一次，状态判定成功的逻辑，放到ReplicationActorList中
  3. Gather 整个 ReplicationActorList
- 使用场景
  1. 每个 UNetReplicationGraphConnection 会有一个 UReplicationGraphNode_TearOff_ForConnection

