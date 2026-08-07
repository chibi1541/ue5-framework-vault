---
related:
  - "[[UEditorEngine/UEditorEngine.Tick|UEditorEngine::Tick]]"
  - "[[Enum/ELevelTick|ELevelTick]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[Enum/ELevelCollectionType|ELevelCollectionType]]"
  - "[[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]]"
  - "[[Enum/ETickingGroup|ETickingGroup]]"
  - "[[FTickTaskManager/FTickTaskManager.EndFrame|FTickTaskManager::EndFrame]]"
tags:
  - LevelTick_cpp
---

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
            SetupPhysicsTickFunctions(DeltaSeconds);

            TickGroup = TG_PrePhysics; // reset this to the start tick group

            FTickTaskManagerInterface::Get().StartFrame(this, DeltaSeconds, TickType, LevelsToTick);

            {
                RunTickGroup(TG_PrePhysics);
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

## 설명
- 월드 단위 tick의 본체. 에디터 월드는 최소한의 tick(파티클 정도)만 돌기 때문에, 실제 흐름은 PIE 월드에서 `LEVELTICK_All`로 호출되는 경우를 기준으로 보면 된다.
- 전체가 `FWorldDelegates::OnWorldTickStart` / `OnWorldTickEnd` 델리게이트로 감싸여 있다.
- `bDoingActorTicks`는 `TickType != LEVELTICK_TimeOnly`일 때만 true다. 즉 [[Enum/ELevelTick|ELevelTick]]이 `LEVELTICK_TimeOnly`면 액터 tick이 돌지 않는다.
- `bValidateLevelList`는 최적화용 플래그다. `DynamicSourceLevels` 외의 컬렉션에 레벨이 하나도 없으면 검증(`Levels.Contains(CollectionLevel)`)을 건너뛰고 모든 레벨을 tick한다.
- [[FLevelCollection/FLevelCollection|FLevelCollection]]을 순회하며 컬렉션별로 tick한다. 오브젝트가 static/dynamic([[Enum/ELevelCollectionType|ELevelCollectionType]])으로 분류되어 있고, 컬렉션마다 tick 시점이 다르다(dynamic → static). static 컬렉션은 Static/Stationary 모빌리티 위주라 tick 대상이 거의 없다.
- 컬렉션의 레벨 중 실제로 월드의 `Levels` 배열에 올라와 있는 것만 `LevelsToTick`으로 모은다. 컬렉션에는 현재 로드되지 않은 레벨이 들어 있을 수 있기 때문.
- [[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]]로 현재 tick 중인 컬렉션 인덱스를 월드에 설정한다. RAII라 스코프를 벗어나면 원래 값으로 복원된다.
- tick 순서는 [[Enum/ETickingGroup|ETickingGroup]]을 따른다: `SetupPhysicsTickFunctions()` → `StartFrame()` → `TG_PrePhysics` → `TG_StartPhysics` → `TG_DuringPhysics`(`bBlockUntilComplete == false`) → `TG_EndPhysics` → `TG_PostPhysics` → `TG_PostUpdateWork` → `TG_LastDemotable` → [[FTickTaskManager/FTickTaskManager.EndFrame|EndFrame()]].
- `TG_DuringPhysics`만 완료를 기다리지 않고 넘어간다. async tick이 다 끝나기 전에 post-physics 작업을 시작해도 되기 때문.
- tick group 사이사이에 LatentActionManager, TimerManager, TickableGameObjects, CameraManager, streaming volume 등의 tick이 함께 돈다.
