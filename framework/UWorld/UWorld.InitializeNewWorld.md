---
related:
  - "[[UWorld/UWorld.CreateWorld|UWorld::CreateWorld]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]"
  - "[[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
tags:
  - World_cpp
---

```cpp
void InitializeNewWorld(const InitializationValues IVS = InitializationValues(), bool bInSkipInitWorld = false)
{
    if (!IVS.bTransactional)
    {
        // haker: clear ObjectFlags in UObjectBase (bit-wise operation)
        // 요건 내부를 한번 따라 쳐보자(선생님이 C++ 플머라면 이건 자유자재로 해야 한다 하심)
        // 내부적으로는 OldFlag & ~FlagToClear하는 방식으로 진행
        ClearFlags(RF_Transactional);
    }
  
    // haker: create default persistent level for new world
    // NewObject로 객체 생성 시에 반드시 Outer를 넣어주어야 함
    // UWorld랑 PersistentLevel은 1:1 관계
    // 월드가 바뀌지 않는 이상 내려가지 않는, 스트리밍 되지 않는 레벨이 PersistentLevel
    // 그렇다는건 UWorld와 PersistentLevel은 같은 UPackage 안에 직렬화 된다는 의미
    // Sub Level들은 별도의 UPackage에 묶임 다만 World Partition은 Sub Level이 존재하지 않기 때문에 이런 방식과는 다르다고 함
    PersistentLevel = NewObject<ULevel>(/*Outer*/this, TEXT("PersistentLevel"));
    PersistentLevel->OwningWorld = this;
  
	  // 시차* : 새로 추가한 구문
	  // UModel이 뭐지? 
	  // UModel : BSP 브러쉬의 지오메트리 정보
	  PersistentLevel->Initialize(FURL(nullptr));
		PersistentLevel->Model = NewObject<UModel>(PersistentLevel);
		PersistentLevel->Model->Initialize(nullptr, 1);
  
    // create the WorldInfo actor
    // haker: we are NOT looking into AWorldSettings in detail, but try to understand it with the example
    // [ ] explain AWorldSettings in the editor
    // 에디터에서 보이는 그 월드 셋팅임, 그걸 액터로 만들어서 스폰함
    AWorldSettings* WorldSettings = nullptr;
	  {
			  // 액터를 스폰할 때 사용하는 파라미터
        // haker: when you spawn an Actor, you need to pass FActorSpawnParameters
        FActorSpawnParameters SpawnInfo;
        SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    
        // set constant name for WorldSettings to make a network replication work between new worlds on host and client
        // haker: you can override WorldSettings' class using UEngine's WorldSettingClass
        SpawnInfo.Name = GEngine->WorldSettingsClass->GetFName();

        // haker: we'll cover SpawnActor in the future (I just leave it in to-do list)
        // 현재 월드의 Persistent Level에 종속된 형태로 스폰
        WorldSettings = SpawnActor<AWorldSettings>(GEngine->WorldSettingsClass, SpawnInfo);
    }
    
    // allow the world creator to override the default game mode in case they do not plan to load a level
    if (IVS.DefaultGameMode)
    {
        WorldSettings->DefaultGameMode = IVS.DefaultGameMode;
    }
    
    // haker: persistent level creates AWorldSettings which contain world info like GameMode
	  PersistentLevel->SetWorldSettings(WorldSettings);
	  
#if WITH_EDITOR
    WorldSettings->SetIsTemporaryHiddenInEditor(true);

    if (IVS.bCreateWorldParitition)
    {
        // haker: we skip world partition for now
        // - BUT, lyra is based on World-Partition
    }
#endif

    if (!bInSkipInitWorld)
    {
        // initialize the world
        InitWorld(IVS);

        // update components
        const bool bRerunConstructionScripts = !FPlatformProperties::RequiresCookedData();
        UpdateWorldComponents(bRerunConstructionScripts, false);
    }
}
```

## 설명
- [[UWorld/UWorld.CreateWorld|UWorld::CreateWorld]]에서 호출되는 실제 초기화 로직. `PersistentLevel`을 `NewObject<ULevel>(this, ...)`로 생성하고 `OwningWorld`를 자기 자신으로 설정한다. `UWorld`와 `PersistentLevel`은 1:1 관계이며 같은 `UPackage`에 직렬화된다.
- `AWorldSettings`를 [[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]로 스폰해 `PersistentLevel`에 종속시킨다. 이 액터가 GameMode 등 월드 정보를 담는다.
- `bInSkipInitWorld`가 false면 `InitWorld`/`UpdateWorldComponents`까지 이어서 호출한다.
