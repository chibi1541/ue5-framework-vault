---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[Enum/ELevelCollectionType|ELevelCollectionType]]"
  - "[[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
tags:
  - World_cpp
---

```cpp
FLevelCollection* FindCollectionByType(const ELevelCollectionType InType)
{
    for (FLevelCollection& LC : LevelCollections)
    {
        if (LC.GetType() == InType)
        {
            return &LC;
        }
    }
    return nullptr;
}
```

## 설명
- `LevelCollections`를 순회하며 `InType`과 일치하는 [[FLevelCollection/FLevelCollection|FLevelCollection]]을 찾는다. [[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]와 [[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]] 양쪽에서 호출된다.
- 소스의 마커 이름은 `FindCollectionIndexByType`이지만 실제 정의된 함수명은 `FindCollectionByType`이다(소스 원문 그대로 유지).
