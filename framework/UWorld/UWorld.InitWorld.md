---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]"
  - "[[UWorld/UWorld.InitializeSubsystems|UWorld::InitializeSubsystems]]"
  - "[[UWorld/UWorld.GetWorldSettings|UWorld::GetWorldSettings]]"
  - "[[UWorld/UWorld.GetDefaultPhysicsVolume|UWorld::GetDefaultPhysicsVolume]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
  - "[[UWorld/UWorld.PostInitializeSubsystems|UWorld::PostInitializeSubsystems]]"
tags:
  - World_cpp
---

```cpp
/** initializes the world, associates the persistent level and sets the proper zones */
void InitWorld(const FWorldInitializationValues IVS = FWorldInitializationValues())
{
    // haker: CoreUObjectDelegate has delegates related to GC events -> USEFUL!
    // FCoreUObjectDelegates가 나왔다, 여긴 UObject, GC관련된 유용한 델리게이트들이 많다고 함
    FCoreUObjectDelegates::GetPostGarbageCollect().AddUObject(this, &UWorld::OnPostGC);

    // haker: we initialize UWorldSubsystems
    InitializeSubsystems();
    
    FWorldDelegates::OnPreWorldInitialization.Broadcast(this, IVS);
  
	  // 이미 Persistent Level에 생성한 WorldSettings을 가져옴
    // 여기서 필요한 이유 : PhysicsScene을 구성할 때 필요 셋팅을 AWorldSettings이 가지기 때문
    AWorldSettings* WorldSettings = GetWorldSettings();
    if (IVS.bInitializeScenes)
    {
        if (IVS.bCreatePhysicsScene)
        {
            // haker: we skip the detail of phsics scene
            // - create physics world
            CreatePhysicsScene(WorldSettings);
        }

        bShouldSimulatePhysics = IVS.bShouldSimulatePhysics;

        // haker: create render world (== FScene)
        // - render world is FScene. we'll cover this later 
        bRequiresHitProxies = IVS.bRequiresHitProxies;
        // 랜더링 월드 생성
        GetRendererModule().AllocateScene(this, bRequiresHitProxies, IVS.bCreateFXSystem, GetFeatureLevel());
    }
    
    // Prepare AI systems
		if (WorldSettings)
		{
			if (IVS.bCreateNavigation || IVS.bCreateAISystem)
			{
				if (IVS.bCreateNavigation)
				{
					FNavigationSystem::AddNavigationSystemToWorld(*this, FNavigationSystemRunMode::InvalidMode, WorldSettings->GetNavigationSystemConfig(), /*bInitializeForWorld=*/false);
				}
				if (IVS.bCreateAISystem && WorldSettings->IsAISystemEnabled())
				{
					CreateAISystem();
				}
			}
		}
    
  
    // haker: add persistent level
    // - you can think of persistent level to have world info(AWorldSettings)
    Levels.Empty(1);
    Levels.Add(PersistentLevel);
    
    PersistentLevel->OwningWorld = this;
    PersistentLevel->bIsVisible = true;
  
    // initialize DefaultPhysicsVolume for the world
    // spawned on demand by this function
    
    // Physics Scene이 없어도 PhysicsVolume은 생성해야 함
    // 그렇지 않으면 콜리전 충돌에 대한 판정이 일어나지 않음(추측에 영역)
    DefaultPhysicsVolume = GetDefaultPhysicsVolume();
  
    // find gravity
    if (GetPhysicsScene())
    {
        FVector Gravity = FVector( 0.f, 0.f, GetGravityZ() );
        GetPhysicsScene()->SetUpForFrame( &Gravity, 0, 0, 0, 0, 0, false);
    }

    // create physics collision handler
    if (IVS.bCreatePhysicsScene)
    {
        //...
	  }
	  
    // haker: from here, UWorld's URL is from Persistnet's URL
    URL = PersistentLevel->URL;
#if WITH_EDITORONLY_DATA || 1
    CurrentLevel = PersistentLevel;
#endif

    // haker: make sure all level collection types are instantiated
    ConditionallyCreateDefaultLevelCollections();
  
    // we're initialized now:
    bIsWorldInitialized = true;
    bHasEverBeenInitialized = true;

    FWorldDelegate::OnPostWorldInitialization.Broadcast(this, IVS);

    PersistentLevel->OnLevelLoaded();

    // haker: notify streaming manager that new level is added
    IStreamingManager::Get().AddLevel(PersistentLevel);

    // haker: call PostInitialize() on each UWorldSubsystem
    PostInitializeSubsystems();

    BroadcastLevelsChanged();

    // haker: let's wrap it up what InitWorld() have done:
    // 1. initialize WorldSubsystemCollection
    // 2. allocate(create) render-world(FScene) and physics-world(FPhysScene)
}
```

## 설명
- [[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]에서 호출되는 실질적인 월드 초기화 함수. Subsystem 초기화, scene(render/physics) 생성, AI 시스템 준비, PersistentLevel 등록, DefaultPhysicsVolume 생성, LevelCollection 생성, PostInitialize까지 이어진다.
- 크게 다음 순서로 진행: [[UWorld/UWorld.InitializeSubsystems|UWorld::InitializeSubsystems]] → scene 생성 → [[UWorld/UWorld.GetWorldSettings|UWorld::GetWorldSettings]] → PersistentLevel 등록 → [[UWorld/UWorld.GetDefaultPhysicsVolume|UWorld::GetDefaultPhysicsVolume]] → [[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]] → [[UWorld/UWorld.PostInitializeSubsystems|UWorld::PostInitializeSubsystems]].
- `bIsWorldInitialized`/`bHasEverBeenInitialized`가 true가 되는 시점이 바로 여기.
