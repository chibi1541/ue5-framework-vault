---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
  - "[[Enum/ELevelCollectionType|ELevelCollectionType]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]"
  - "[[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
tags:
  - World_cpp
---

```cpp
/** creates the dynamic source and static level collections if they don't already exists */
void ConditionallyCreateDefaultLevelCollections()
{
    LevelCollections.Reserve((int32)ELevelCollectionType::MAX);

    // create main level collection; the persistent level will always be considered dynamic
    if (!FindCollectionByType(ELevelCollectionType::DynamicSourceLevels))
    {
        // default to the dynamic source collection
        // Persistent Level은 dynamic source collection?
        ActiveLevelCollectionIndex = FindOrAddCollectionByType_Index(ELevelCollectionType::DynamicSourceLevels);

        // haker: dynamc/static level collections are set its persistent level for World's persistent level
        // 항상 World에서 생성한 Persistent Level을 설정함
        LevelCollections[ActiveLevelCollectionIndex].SetPersistentLevel(PersistentLevel);

        // don't add the persistent level if it is already a member of another collection
        // this may be the case if, for example, this world is the outer of a streaming level,
        // in which case the persistent level may be in one of the collections in the streaming level's OwningWorld
        if (PersistentLevel->GetCachedLevelCollection() == nullptr)
        {
            // haker: if persistent level is not set, defaultly add it to the dynamic level collection
            LevelCollections[ActiveLevelCollectionIndex].AddLevel(PersistentLevel);
        }
    }

    if (!FindCollectionByType(ELevelCollectionType::StaticLevels))
    {
        FLevelCollection& StaticCollection = FindOrAddCollectionByType(ELevelCollectionType::StaticLevels);
        StaticCollection.SetPersistentLevel(PersistentLevel);
    }
}
```

## 설명
- [[UWorld/UWorld.InitWorld|UWorld::InitWorld]]에서 호출. `DynamicSourceLevels`/`StaticLevels` [[FLevelCollection/FLevelCollection|FLevelCollection]]이 아직 없으면 [[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]로 생성하고 `PersistentLevel`을 등록한다.
- `PersistentLevel`은 항상 dynamic source collection에 속하는 것으로 취급되며, 이미 다른 컬렉션에 속해 있지 않을 때만 추가한다(스트리밍 레벨의 outer인 경우 등 예외 있음).
