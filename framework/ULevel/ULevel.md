---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[AActor/AActor|AActor]]"
  - "[[UObject/UObject|UObject]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
  - "[[UWorld/UWorld.GetWorldSettings|UWorld::GetWorldSettings]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[UActorComponent/UActorComponent.SetupActorComponentTickFunction|UActorComponent::SetupActorComponentTickFunction]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[FTickTaskManager/FTickTaskManager.TickTaskLevelForLevel|FTickTaskManager::TickTaskLevelForLevel]]"
  - "[[ULevel/ULevel.UpdateLevelComponents|ULevel::UpdateLevelComponents]]"
  - "[[ULevel/ULevel.IncrementalUpdateComponents|ULevel::IncrementalUpdateComponents]]"
  - "[[ULevel/ULevel.IncrementalRegisterComponents|ULevel::IncrementalRegisterComponents]]"
  - "[[SortActorsHierarchy|SortActorsHierarchy]]"
  - "[[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
tags:
  - Level_h
---

```cpp
/**
 * the level object:
 * contains the level's actor list, BSP information, and brush list
 * every level has a World as its Outer and can be used as the PersistentLevel, however,
 * when a Level has been streamed in the OwningWorld represents the World that it is a part of 
 */

/**
 * a level is a collection of Actors (lights, volumes, mesh instances etc)
 * multiple levels can be loaded and unloaded into the World to create a streaming experience 
 */

// haker:
// Level?
// - level == collection of actors:
//   - examples of actors:
//     - light, static-mesh, volume, brush(e.g. BSP brush: binary-search-partitioning), ...
//       [ ] explain BSP brush with the editor
// - rest of content will be skipped for now:
//   - when we cover different topics like world-partition, streaming etc, we will visit it again
class ULevel : public UObject
{
		/** array of all actors in this level, used by FActorIteratorBase and derived classes */
		// haker: this is the member variable which contains a list of AActor
		TArray<TObjectPtr<AActor>> Actors;
		
		/** cached level collection that this level is contained in */
		FLevelCollection* CachedLevelCollection;
}
```

```cpp
class ULevel : public UObject
{
		// ...

		// 이게 월드가 아닌 레벨이 액터를 들고 관리하는 이유?
		/** cached level collection that this level is contained in */
		FLevelCollection* CachedLevelCollection;
		
		// 레벨을 스트리밍 할 때 증분적으로 처리하기 위해 나눈 구간 타입
		enum class EIncrementalComponentState : uint8
		{
		    Init,
		    RegisterInitialComponents,
		#if WITH_EDITOR || 1
				// 게임에서는 이미 쿠킹된 형태이므로 이 단계가 불릴 필요가 없어서 에디터 온리
		    RunConstructionScripts,
		#endif
		    Finalize,
		};
		
		/** the current stage for incrementally updating actor components in the level */
		// haker: we already covered AActor's initialization steps
		EIncrementalComponentState IncrementalComponentState;
		
		// 여기도 각 단계를 체크하기 위한 비트 플래그
		/** whether the actor referenced by CurrentActorIndexForUpdateComponents has called PreRegisterAllComponents */
		uint8 bHasCurrentActorCalledPreRegister : 1;
		
		/** whether components are currently registered or not */
		uint8 bAreComponentsCurrentlyRegistered : 1;
		
		// RegisterInitialComponents 단계 내부에서 증분적으로 진행된 단계를 체크하는 변수
		/** current index into actors array for updating components */
		// haker: tracking actor index in ULevel's ActorList to support incremental update
		int32 CurrentActorIndexForIncrementalUpdate;
		
		// 이건 나중에 Tick에서 한번에 확인...
		/** data structures for holding the tick functions */
		// haker: for now, member variables related to tick function are skipped
		FTickTaskLevel* TickTaskLevel;
}
```

## 설명
- 레벨은 액터의 집합(light, static-mesh, volume, BSP brush 등)이다. 액터 리스트, BSP 정보, brush 리스트를 갖는다.
- 모든 레벨은 `UWorld`를 Outer로 가진다. `PersistentLevel`로 쓰일 수도 있고, 스트리밍된 레벨의 경우 `OwningWorld`가 자신이 속한 월드를 가리킨다.
- 여러 레벨을 월드에 load/unload 하면서 streaming 경험을 만든다.
- `Actors`가 이 레벨의 모든 액터를 담고 있으며 `FActorIteratorBase` 계열이 이를 순회한다.
- `CachedLevelCollection`은 이 레벨이 속한 [[FLevelCollection/FLevelCollection|FLevelCollection]]을 캐싱한 것.
- `IncrementalComponentState`(`EIncrementalComponentState`)는 레벨 스트리밍 시 액터 컴포넌트를 증분적으로 등록하기 위한 단계 구분이다(`Init` → `RegisterInitialComponents` → (에디터 한정) `RunConstructionScripts` → `Finalize`).
- `CurrentActorIndexForIncrementalUpdate`는 `RegisterInitialComponents` 단계 내부에서 증분 진행 중인 액터 인덱스를 추적한다.
- `TickTaskLevel`은 틱 함수 관련 데이터 구조로, 자세한 내용은 추후 다룬다.
