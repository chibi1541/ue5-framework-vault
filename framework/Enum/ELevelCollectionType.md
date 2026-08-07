---
related:
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
  - "[[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]"
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
tags:
  - EngineTypes_h
---

```cpp
/** indicates the type of a level collection, used in FLevelCollection */
// haker: don't sticking to its types, we are going to understand it simply like dynamic level vs. static level
// 레벨을 분류할 때 제한 요소를 추가해서 최적화 여지를 두기 위한 타입
enum class ELevelCollectionType : uint8
{
		// DynamicSourceLevels, DynamicDuplicatedLevels를 묶어서 DynamicLevel
    /**
     * the dynamic levels that are used for normal gameplay and the source for any duplicated collections
     * will contain a world's persistent level and any streaming levels that contain dynamic or replicated gameplay actors
     */
    DynamicSourceLevels,

    /** gameplay relevant levels that have been duplicated from DynamicSourceLevels if requested by the game */
    DynamicDuplicatedLevels,

    /**
     * these levels are shared between the source levels and the duplicated levels, and should contain
     * only static geometry and other visuals that are not replicated or affected by gameplay
     * thsese will not be duplicated in order to save memory 
     */
    StaticLevels,

    MAX
};
```

## 설명
- [[FLevelCollection/FLevelCollection|FLevelCollection]]을 분류하는 타입. 구체적인 항목보다는 dynamic level(게임플레이용, 복제됨) vs. static level(공유되는 정적 지오메트리/비주얼, 복제 안 됨) 구분으로 이해하면 된다.
- `DynamicSourceLevels`/`DynamicDuplicatedLevels`는 묶어서 dynamic level, `StaticLevels`는 메모리 절약을 위해 복제되지 않고 공유된다.
