---
related:
  - "[[UWorld/UWorld.SetActiveLevelCollection|UWorld::SetActiveLevelCollection]]"
  - "[[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld|UWorld]]"
tags:
  - World_cpp
---

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

## 설명
- `ActiveLevelCollectionIndex`로 `LevelCollections`를 인덱싱해 현재 컨텍스트가 설정된 [[FLevelCollection/FLevelCollection|FLevelCollection]]을 돌려준다. 유효하지 않은 인덱스(`INDEX_NONE` 포함)면 nullptr.
- 반환값이 nullptr이 아니라는 것은 지금 이 월드에 대한 [[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]] 스코프 안에서 실행 중이라는 뜻이다.
