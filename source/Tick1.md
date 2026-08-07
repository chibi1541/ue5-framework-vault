---
summarize: true
---

# UEditorEngine::Tick

#### Foundation - Tick - UEditorEngine::Tick(EditorEngine.cpp)

```cpp
virtual void Tick(float DeltaSeconds, bool bIdleMode) override
{
    // haker: where does GWorld is updated initially?
    UWorld* CurrentGWorld = GWorld;

    FWorldContext& EditorContext = GetEditorWorldContext();
    
    // viewport 관련 초기화 처리
    
    // by default we tick the editor world
    bool bShouldTickEditorWorld = true;

    ELevelTick TickType = IsRealTfime ? LEVELTICK_ViewportOnly : LEVELTICK_TimeOnly;
    // PIE가 아닌 에디터 월드에서는 파티클 정도만 Tick이 돔
    // 실제 Tick이 돌려면 PIE 환경이 되어야 함
    if (bShouldTickEditorWorld)
    {
        // NOTE: still allowing the FX system to tick so particle systems don't restart after entering/leaving responsive mode
        // haker: FX(niagara) system should be running even in level-viewport mode
        {
            // haker: here is EditorWorld for level-viewport ticking:
            // - level-viewport's world is ticking in very limited allowance
            // - so, we look into world's tick with PIE Context
            // 지금 이건 Editor World임 밑에 PIE World가 있음
            EditorContext.World()->Tick(TickType, DeltaSeconds);
        }
    }

    {
        // determine number of PIE worlds that should tick 
        TArray<FWorldContext*> LocalPieContextPtrs;
        for (FWorldContext& PieContext : WorldList)
        {
            // haker: EWorldType::PIE!
            if (PieContext.WorldType == EWorldType::PIE && PieContext.World() != nullptr)
            {
                LocalPieContextPtrs.Add(&PieContext);
            }
        }

				// CreateInnerProcessPIEGameInstance()는 나중에 확인 
        // haker: when is WorldContext for PIE initialized? 
        // - when you want to know exactly what happen on this: see the callstack after setting BP(breakpoint) to CreateInnerProcessPIEGameInstance
        // see UEditorEngine::CreateInnerProcessPIEGameInstance()
        // - for now, we are looking into how tick function runs first, then we go back to see how PIE is initialized

        for (FWorldContext* PieContextPtr : LocalPieContextPtrs)
        {
            // haker: after we see CreateInnerProcessPIEGameInstance(), then we understand what GameViewport is for
            FWorldContext& PieContext = *PieContextPtr;
            PlayWorld = PieContext.World();
            GameViewport = PieContext.GameViewport;

            /** how much time to use per tick */
            // haker: nothing special, it allows to run in fixed fps with repsect to DeltaSeconds
            float TickDeltaSeconds;
            if (PieContext.PIEFixedTickSeconds > 0.f)
            {
                PieContext.PIEAccumulatedTickSeconds += DeltaSeconds;
                TickDeltaSeconds = PieContext.PIEFixedTickSeconds;
            }
            else
            {
                // haker: otherwise, we will get into here (one tick per frame)
                PieContext.PIEAccumulatedTickSeconds = DeltaSeconds;
                TickDeltaSeconds = DeltaSeconds;
            }

            for (; PieContext.PIEAccumulatedTickSeconds >= TickDeltaSeconds; PieContext.PIEAccumulatedTickSeconds -= TickDeltaSeconds)
            {
                // update the level
                {
                    // tick the level
                    PieContext.World()->Tick(LEVELTICK_All, TickDeltaSeconds);
                }
            }
        }
    }
}
```

#### Foundation - Tick - UEditorEngine::GetEditorWorldContext(EditorEngine.cpp)

```cpp
FWorldContext& UEditorEngine::GetEditorWorldContext(bool bEnsureIsGWorld)
{
	// 월드 생성 시에 Persistent Level을 만드므로 하나 이상은 반드시 있음
	// WorldList는 TIndirectArray 이므로 FWorldContext의 포인터를 저장함
	// 하지만 사용 방식 자체는 TArray와 같기 때문에 인덱스 연산자([])가 포인터 내부의 값을 반환함
	for (int32 i=0; i < WorldList.Num(); ++i)
	{
		if (WorldList[i].WorldType == EWorldType::Editor)
		{
			ensure(!bEnsureIsGWorld || WorldList[i].World() == GWorld);
			return WorldList[i];
		}
	}

	check(false); // There should have already been one created in UEngine::Init
	return CreateNewWorldContext(EWorldType::Editor);
}
```

#### Foundation - Tick - enum ELevelTick(EngineBaseType.h)

```cpp
/** type of tick we wish to perform on the level */
enum ELevelTick
{
    /** update the level time only */
	  // 액터의 Tick(게임 로직)은 돌지 않고 게임 시간만 흐름
    LEVELTICK_TimeOnly = 0,
    /** update time and viewports */
    // 그래픽적인 요소(카메라, 랜더링)만 Tick 이벤트가 진행
    LEVELTICK_ViewportsOnly = 1,
    /** update all */
    // 모든 게임 루프가 진행
    LEVELTICK_All = 2,
    /** delta time is zero, we are paused; components don't tick */
    // 모든 시간이 정지(게임 시간 자체가 흐르지 않음)
    LEVELTICK_PauseTick = 3,
};
```

# UWorld::Tick

#### Foundation - Tick - UWorld::Tick(LevelTick.cpp)

에디터 월드는 기본적으로 필요 최소한의 Tick만 돌기 때문에(파티클 같은 것들) 여기서는 PIE 환경에서 Tick을 돈다고 생각

```cpp
/**
 * update the level after a variable amount of time, DeltaSeconds, has passed
 * all child actors are ticked after their owners have been ticked 
 */
void Tick(ELevelTick TickType, float DeltaSeconds)
{
		// 유용한 델리게이트들로 감싸져있음 (OnWorldTickStart, OnWorldTickEnd) 
    FWorldDelegates::OnWorldTickStart.Broadcast(this, TickType, DeltaSeconds);

		// 경과 시간 계산

		// LEVELTICK_TimeOnly 이 상황에서는 Actor Tick이 돌지 않음
    bool bDoingActorTicks = 
        (TickType != LEVELTICK_TimeOnly);

		// what is level collection : static level, dynamic level(persistent level은 이쪽)
    // 왜 level collection가 필요한가?
    // if only the DynamicLevel collection has entries, we can skip the validation and tick all levels
    // haker: it just minor optimization to skip the code: Levels.Contains(CollectionLevel)
    bool bValidateLevelList = false;
    for (const FLevelCollection& LevelCollection : LevelCollections)
    {
		    // ELevelCollectionType::DynamicSourceLevels 일 경우에
		    // Tick을 도는 레벨을 체크함
        if (LevelCollection.GetType() != ELevelCollectionType::DynamicSourceLevels)
        {
            const int32 NumLevels = LevelCollection.GetLevels().Num();
            if (NumLevels != 0)
            {
                bValidateLevelList = true;
                break;
            }
        }
    }

    // haker: we can understand what FLevelCollection for:
    // - object can be categorized into static/dynamic
    // - dynamic or static objects are into different category (LevelCollection)
    // - each level collection ticks on different moment (dynamic -> static)
    for (int32 i = 0; i < LevelCollections.Num(); ++i)
    {
		    // Static Level의 경우 모빌리티가 Static이거나 Stationary만 모아놨을 것이기 때문에
		    // Tick을 도는 개체가 없거나 거의 없음
		    // Tick을 도는 객체를 Level 단위로 분류에 collection에 넣어뒀다? 정도로 생각하면 좋을듯?
        // build a list of levels from the collection that are also in the world's level array
        // collections may contain levels that aren't loaded in the world at the moment
        TArray<ULevel*> LevelsToTick;
        for (ULevel* CollectionLevel : LevelCollections[i].GetLevels())
        {
            // haker: here!
            const bool bAddToTickList = (bValidateLevelList == false) || Levels.Contains(CollectionLevel);
            if (bAddToTickList && CollectionLevel)
            {
                LevelsToTick.Add(CollectionLevel);
            }
        }

        // set up context on the world for this level collection
        // haker: update active level collection by level collection's index
        FScopedLevelCollectionContextSwitch LevelContext(i, this);

        // if caller wants time update only, or we are paused, skip the rest
        const bool bShouldSkipTick = (LevelsToTick.Num() == 0);
        if (bDoingActorTicks && !bShouldSkipTick)
        {
            // actually tick actors now that context is set up
            // haker: understand what SetupPhysicsTickFunctions() does for us:
            ***[SetupPhysicsTickFunctions](<https://app.notion.com/p/UWorld-SetupPhysicsTickFunctions-39339b66a33d8068a0c6e87b8b4f47cc?pvs=21>)(***DeltaSeconds);

            TickGroup = TG_PrePhysics; // reset this to the start tick group

            FTickTaskManagerInterface::Get().[***StartFrame***](<https://app.notion.com/p/FTickTaskManager-StartFrame-39339b66a33d809fa05bf854f318bd7b?pvs=21>)(this, DeltaSeconds, TickType, LevelsToTick);

            {
                [***RunTickGroup***](<https://app.notion.com/p/UWorld-RunTickGroup-39339b66a33d8082a385c7a84ae41a64?pvs=21>)(TG_PrePhysics);
            }
            {
                RunTickGroup(TG_StartPhysics);
            }
            {
                // no wait here, we should run until idle though
                // we don't care if all of the async ticks are done before we start running post-phys stuff
                // haker: note that bBlockUntilComplete == false
                RunTickGroup(TG_DuringPhysics, false);
            }
            {
                // set this here so the current tick group is correct during collision notifies, though I am not sure it matters 
                // 'cause of the false up there'
                TickGroup = TG_EndPhysics;
                RunTickGroup(TG_EndPhysics);
            }
            {
                RunTickGroup(TG_PostPhysics);
            }
        }

        // various ticks:
        // 1. LatentActionManager
        // 2. TimerManager
        // 3. TickableGameObjects
        // 4. CameraManager
        // 5. streaming volume

        if (bDoingActorTicks)
        {
            RunTickGroup(TG_PostUpdateWork);
            RunTickGroup(TG_LastDemotable);
        }

        FTickTaskManagerInterface::Get().EndFrame();
    }

    FWorldDelegates::OnWorldTickEnd.Broadcast(this, TickType, DeltaSeconds);
}
```

#### Foundation - Tick - class FScopedLevelCollectionContextSwitch (World.h)

Scoped라는 키워드가 들어가있는 경우 RAII 패턴이 들어가 있기 때문에 생성자, 소멸자 중심으로 보면 좋음

내부적으로는 UWorld의 CurrentLevel을 InLevelCollectionIndex에 해당하는 LevelCollection의 PersistentLevel로 교체한 후에 FScopedLevelCollectionContextSwitch가 메모리 해제되면 원래대로 돌려 놓음

```cpp
// haker: RAII (Resource Acquisition is initialization) pattern:
// - change ActiveLevelCollection in UWorld
class FScopedLevelCollectionContextSwitch
{
public:
    /**
     * constructor that will save the current relevant values of InWorld
     * and set the collection's context values for InWorld 
     */
    FScopedLevelCollectionContextSwitch(int32 InLevelCollectionIndex, UWorld* const InWorld)
        : World(InWorld)
        , SavedTickingCollectionIndex(InWorld ? InWorld->GetActiveLevelCollectionIndex() : INDEX_NONE)
    {
        if (World)
        {
            World->SetActiveLevelCollection(InLevelCollectionIndex);
        }
    }

    /** the destructor restores the context on the world that was saved in the constructor */
    ~FScopedLevelCollectionContextSwitch()
    {
        if (World)
        {
            World->SetActiveLevelCollection(SavedTickingCollectionIndex);
        }
    }

private:
    class UWorld* World;
    int32 SavedTickingCollectionIndex;
};
```

#### Foundation - Tick - UWorld::SetActiveLevelCollection(World.cpp)

```cpp
/** sets the level collection and its context on this world. should only be called by FScopedLevelCollectionContextSwitch */
void SetActiveLevelCollection(int32 LevelCollectionIndex)
{
    ActiveLevelCollectionIndex = LevelCollectionIndex;

    // see GetActiveLevelCollection()
    const FLevelCollection* const ActiveLevelCollection = GetActiveLevelCollection();

    // haker: if ActiveLevelCollectionIndex is INDEX_NONE, it will have nullptr
    if (ActiveLevelCollection == nullptr)
    {
        return;
    }

    // haker: as we saw last week, all level collection has same persistent level right?
    // 여기의 PersistentLevel은 UWorld의 PersistentLevel
    PersistentLevel = ActiveLevelCollection->GetPersistentLevel();
    if (IsGameWorld())
    {
        SetCurrentLevel(ActiveLevelCollection->GetPersistentLevel());
    }

    // haker: it actually overrides NetDriver and etc.
    // 이 이외도 다양한 처리들이 일어나지만 여기서 알아야 하는건
    // 해당 LevelCollection의 PersistentLevel를 CurrentLevel로 재설정 해준다 정도
}
```

#### Foundation - Tick - UWorld::GetActiveLevelCollection(World.cpp)

```cpp
/**
 * returns the level collection which currently has its context set on this world. may be null.
 * if non-null, this implies that execution is currently within the scope of an FScopedLevelCollectionContextSwitch for this world.
 */
const FLevelCollection* GetActiveLevelCollection() const
{
    if (LevelCollections.IsValidIndex(ActiveLevelCollectionIndex))
    {
        return &LevelCollections[ActiveLevelCollectionIndex];
    }
    return nullptr;
}
```

## TickTaskManager::EndFrame

#### Foundation - Tick - FTickTaskManager::EndFrame(TickTaskManager.cpp)

```cpp
/** Finish a frame of ticks **/
virtual void EndFrame() override
{
	TickTaskSequencer.EndFrame();
	bTickNewlySpawned = false;
	for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
	{
		LevelList[LevelIndex]->EndFrame();
	}

	FTaskSyncManager* SyncManager = FTaskSyncManager::Get();
	if (SyncManager)
	{
		SyncManager->EndFrame(Context.World);
	}

	Context.World = nullptr;
	LevelList.Reset();
}
```

#### Foundation - Tick - FTickTaskLevel::EndFrame(TickTaskManager.cpp)

```cpp
/** end a tick frame */
void EndFrame()
{
		// 마지막에 RescheduleList, CoolingDownList를 업데이트
    ScheduleTickFunctionCooldowns();

    // haker: note that TickNewlySpawned is set as false
    bTickNewlySpawned = false;
}
```
