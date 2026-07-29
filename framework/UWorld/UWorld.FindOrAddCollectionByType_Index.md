---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
tags:
  - World_cpp
---

```cpp
int32 FindOrAddCollectionByType_Index(const ELevelCollectionType InType)
{
    const int32 FoundIndex = FindCollectionIndexByType(InType);

    if (FoundIndex != INDEX_NONE)
    {
        return FoundIndex;
    }

    // Not found, add a new one.
    FLevelCollection NewLC;
    NewLC.SetType(InType);
    return LevelCollections.Add(MoveTemp(NewLC));
}
```

## 설명
- [[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]에서 호출. [[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]으로 먼저 찾아보고, 없으면 새 [[FLevelCollection/FLevelCollection|FLevelCollection]]을 만들어 `LevelCollections`에 추가한 뒤 그 인덱스를 반환한다.
