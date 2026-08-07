---
related:
  - "[[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]]"
  - "[[UWorld/UWorld.GetActiveLevelCollection|UWorld::GetActiveLevelCollection]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[UWorld/UWorld|UWorld]]"
tags:
  - World_cpp
---

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

## 설명
- `ActiveLevelCollectionIndex`를 갱신하고, 그 컬렉션의 `PersistentLevel`을 월드의 `PersistentLevel`(게임 월드라면 `CurrentLevel`까지)로 재설정한다.
- 주석대로 [[FScopedLevelCollectionContextSwitch/FScopedLevelCollectionContextSwitch|FScopedLevelCollectionContextSwitch]]에서만 호출해야 하는 함수다.
- 인덱스가 `INDEX_NONE`이면 [[UWorld/UWorld.GetActiveLevelCollection|GetActiveLevelCollection()]]이 nullptr를 반환하므로 인덱스만 갱신하고 빠져나온다.
- NetDriver 등 다른 컨텍스트도 함께 덮어쓰지만, 핵심은 해당 컬렉션의 `PersistentLevel`을 현재 레벨로 바꿔주는 것이다.
