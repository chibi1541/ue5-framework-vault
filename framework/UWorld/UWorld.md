---
related:
  - "[[UWorld/UWorld.CreateWorld|UWorld::CreateWorld]]"
  - "[[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UObject/UObject|UObject]]"
  - "[[Enum/EWorldType|EWorldType]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
  - "[[UWorld/UWorld.InitializeSubsystems|UWorld::InitializeSubsystems]]"
  - "[[UWorld/UWorld.GetWorldSettings|UWorld::GetWorldSettings]]"
  - "[[UWorld/UWorld.GetDefaultPhysicsVolume|UWorld::GetDefaultPhysicsVolume]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
  - "[[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]"
  - "[[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]"
  - "[[UWorld/UWorld.PostInitializeSubsystems|UWorld::PostInitializeSubsystems]]"
tags:
  - World_h
---

```cpp
/** 
 * the world is the top level object representing a map or a sandbox in which Actors and Components will exist and be rendered 
 * 
 * a world can be a single persistent level with an optional list of streaming levels that are loaded and unloaded via volumes and blueprint functions
 * or it can be a collection of levels organized with a World Composition (->haker: OLD COMMENT...)
 * 
 * in a standalone game, generally only a single World exists except during seamless area transition when both a destination and current world exists
 * in the editor many Worlds exist: 
 * - the level being edited
 * - each PIE instance
 * - each editor tool which has an interactive rendered viewport, and many more
 */
 
class UWorld final : public UObject, public FNetworkNotify
{
		/** the URL that was used when loading this World */
		// haker: think of the URL as package path:
		// - e.g. Game\Map\Seoul\Seoul.umap
		// 파일 Path, 패키지 Path
		FURL URL;
		
		/** the type of world this is. Describes the context in which it is being used (Editor, Game, Preview etc.) */
		// haker: we already seen EWorldType
		// - TEnumAsByte is helper wrapper class to support bit operation on enum type
		// - I recommend to read it how it is implemented
		// ADVICE: as C++ programmer, it is VERY important **to manipulate bit operations freely!**
		// enum 값을 비트 연산하기 위한 랩핑
		TEnumAsByte<EWorldType::Type> WorldType;
		
		/** persistent level containing the world info, default brush and actors pawned during gameplay among other things */
		// hacker: short explanation about world info
		TObjectPtr<class ULevel> PersistentLevel;
		
		/** array of level collections currently in this world */
		// haker: UWorld has the classified collections of level we have covered in ULevel
		TArray<FLevelCollection> LevelCollections;
		
		// 월드 컬렉션을 TickFunction 기준으로 나눌 때 사용. 나중에...
		/** index of the level collection that's currently ticking */
		int32 ActiveLevelCollectionIndex;
		
		// 월드 생성 시, 피직스 월드 생성 -> 이걸 특정 영역에만 적용하고 싶을 경우(얘를 들어 배경에는 굳이 물리 처리가 필요 없을 때)
		// 이 범위를 조정하는 것이 피직스 볼륨
		/** DefaultPhysicsVolume used for whole game */
		// haker: you can think of physics volume as 3d range of physics engine works (== physics world covers)
		TObjectPtr<APhysicsVolume> DefaultPhysicsVolume;
		
		// 그냥 여기 피직스 Scene이 있다는 정도만 확인
		/** physics scene for this world */
		FPhysScene* PhysicsScene;
		
		// FWorldSubsystemCollection == FObjectSubsystemCollection<UWorldSubsystem> 
		// subsystem을 관리하기 위한 함수등이 들어있음
		FWorldSubsystemCollection SubsystemCollection;
		
		
		/** line batchers: */
		// ULineBatchComponent는 PrimitiveComponent 즉 SceneComponent로서 실제 좌표를 가지고
		// 랜더되는 아이라서 Actor 계열에 있어야 하는데 이거 왜 UWorld가 가지고 잇는가?
		// 디버깅 기능으로 추가한 녀석들인데 이렇게 되면 월드의 서브오브젝트로 끼어들어가는 형태라
		// 기본 월드 - 레벨 - 액터 - 컴포넌트 구조와는 어긋남
		// 이런 경우 월드가 초기화 할 때 별도로 초기화 과정을 거침
		// haker: debug lines
		// - ULineBatchComponents are resided in UWorld's subobjects
		// UE_DEPRECATED(5.6, "얘들 다 UWorld::GetLineBatcher(ELineBatcherType Type)로 대체됨")
		// 얘들이 월드 Init하는 단계에서 곧바로 컴포넌트 Register를 수행하기 때문에 조금 특별한 취급
		TObjectPtr<class ULineBatchComponent> LineBatcher;
		TObjectPtr<class ULineBatchComponent> PersistentLineBatcher;
		TObjectPtr<class ULineBatchComponent> ForegroundLineBatcher;
}
```

## 설명
- 액터와 컴포넌트가 존재하고 렌더링되는 map/sandbox를 나타내는 최상위 객체. standalone game에서는 보통 하나만 존재하지만, 에디터에서는 편집 중인 레벨, 각 PIE 인스턴스, 뷰포트를 가진 에디터 툴마다 월드가 존재한다.
- `URL`은 패키지 경로(파일 경로)라고 생각하면 된다. (예: `Game\Map\Seoul\Seoul.umap`)
- `WorldType`은 `TEnumAsByte`로 감싸 enum 값에 비트 연산을 쓸 수 있게 한 것.
- `PersistentLevel`은 World Setting에 나오는 World Info를 들고 있는 레벨. 이를 갖지 않는 나머지 레벨들이 SubLevel이다.
- `LevelCollections`는 [[FLevelCollection/FLevelCollection|FLevelCollection]]의 배열로, `ActiveLevelCollectionIndex`가 현재 틱 중인 컬렉션을 가리킨다.
- `DefaultPhysicsVolume`/`PhysicsScene`은 월드 생성 시 만들어지는 physics 관련 객체. physics volume은 physics 처리가 적용되는 3D 범위로 이해하면 된다.
- `SubsystemCollection`은 `FObjectSubsystemCollection<UWorldSubsystem>` 타입으로, 월드 단위 서브시스템들을 관리한다.
- `LineBatcher` 계열은 디버그 라인을 그리기 위한 서브오브젝트로, 월드-레벨-액터-컴포넌트 계층 구조와는 별도로 월드 초기화 시 직접 등록된다.
