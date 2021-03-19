# Actor 同步
基于版本 4.26.1 分析

之前有写反射相关的，后面提到到同步数据会用到反射信息<br>
网络同步堆栈信息如下（不使用ReplicateGraph）
```
UNetDriver::TickFlush
UNetDriver::ServerReplicateActors
UNetDriver::ServerReplicateActors_ProcessPrioritizedActors
UActorChannel::ReplicateActor()
FObjectReplicator::ReplicateProperties
```
整个Actor的差量复制同步依赖FObjectReplicator
```
TSharedPtr<class FReplicationChangelistMgr> ChangelistMgr;
TSharedPtr<FRepLayout> RepLayout;
TUniquePtr<FRepState>  RepState;
TUniquePtr<FRepState> CheckpointRepState;

UClass* ObjectClass;
```
在Actor关联到 UActorChannel 时会调用 UNetConnection::CreateReplicatorForNewActorChannel 生成 FObjectReplicator 信息
```
TSharedPtr<FObjectReplicator> UNetConnection::CreateReplicatorForNewActorChannel(UObject* Object)
{
	TSharedPtr<FObjectReplicator> NewReplicator = MakeShareable(new FObjectReplicator());
	NewReplicator->InitWithObject( Object, this, true );
	return NewReplicator;
}

void FObjectReplicator::InitWithObject( UObject* InObject, UNetConnection * InConnection, bool bUseDefaultState )
{
    // .....
    // 保存反射信息
	ObjectClass					= InObject->GetClass(); 

    // 生成 FRepLayout 这个是同步的核心
	RepLayout = Connection->Driver->GetObjectClassRepLayout( ObjectClass );

	// Make a copy of the net properties
	uint8* Source = bUseDefaultState ? (uint8*)GetObject()->GetArchetype() : (uint8*)InObject;

	InitRecentProperties( Source );

	Connection->Driver->AllOwnedReplicators.Add(this);
}
```
FRepLayout 这个辅助类是做脏数据的比较和历史同步数据之类的，这个实现特别复杂，这边先大概找到入口 <br>
FRepLayout::CompareProperties 比对是否有差异 <br>
FRepLayout::SendProperties 提取并发送差异 <br>
FRepLayout::BuildSharedSerialization 可共享的序列化数据 <br>
属性类型的 TArray 会特殊处理，不过本质就是递归调用 <br>
TMap 还没发现是什么机制
