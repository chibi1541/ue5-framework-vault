---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[Enum/ELevelCollectionType|ELevelCollectionType]]"
  - "[[UWorld/UWorld.ConditionallyCreateDefaultLevelCollections|UWorld::ConditionallyCreateDefaultLevelCollections]]"
  - "[[UWorld/UWorld.FindOrAddCollectionByType_Index|UWorld::FindOrAddCollectionByType_Index]]"
  - "[[UWorld/UWorld.FindCollectionIndexByType|UWorld::FindCollectionIndexByType]]"
tags:
  - World_h
---

```cpp
// haker: FLevelCollection is collection based on ELevelCollectionType
struct FLevelCollection
{
    /** the type of this collection */
    ELevelCollectionType CollectionType;

    /**
     * the persistent level associated with this collection
     * the source collection and the duplicated collection will have their own instances 
     */
    // haker: usually OwnerWorld's PersistentLevel
    TObjectPtr<class ULevel> PersistentLevel;

    /** all the levels in this collection */
    TSet<TObjectPtr<ULevel>> Levels;
    
    // 왜 outer라는 개념이 있는데도 굳이 OwningWorld를 따로 관리해야하는가?
    // 레벨 컴포지션에서는 레벨의 outer가 반드시 owningworld가 아니라고 함?
    // 이 부분은 디버깅이 필요할 듯
    /**
     * the world that has this level in its Levels array
     * this is not the same as GetOuter(), because GetOuter() for a streaming level is a vestigial world that is not used
     * it should not be accessed during BeginDestroy(), just like any other UObject references, since GC may occur in any order
     */
    // haker: let's understand OwningWorld vs. OuterPrivate
    // - note that my explanation is based on WorldComposition's level streaming or LevelBlueprint's level load/unload manipulation
    //   - World Partition has different concept which usually OwningWorld and OuterPrivate is same
    // - Diagram:                                                                                                        
    //      World0(OwningWorld)──[OuterPrivate]──►Package0(World.umap)                                          
    //       ▲                                                                                                  
    //       │                                                                                                  
    // [OuterPrivate]                                                                                           
    //       │                                                                                                  
    //       │                                                                                                  
    //      Level0(PersistentLevel)-> persistent 레벨은 월드와 1:1                                                                            
    //       │                                                                                                  
    //       │                                                                                                  
    //       ├────Level1──[OuterPrivate]──►World1(왜 여기 월드가 또 있나?, AI에 물어보면 이런거 없다는데?)───[OuterPrivate]───►Package1(Level1.umap)
    //       │    (OwningWorld는 World0)                  
    //       │                                                                                                  
    //       └────Level2───────►World2───────►Package2(Level2.umap) 
    //            (여기도 OwningWorld는 World0)                                              
    TObjectPtr<UWorld> OwningWorld;
};
```

## 설명
- `ELevelCollectionType`에 따라 레벨들을 분류해 묶어놓은 컬렉션. `UWorld`의 `LevelCollections` 배열이 이 구조체들의 배열이다.
- `PersistentLevel`/`Levels`는 이 컬렉션에 속한 레벨들, `OwningWorld`는 이 컬렉션을 담고 있는 월드.
- `OwningWorld`는 `GetOuter()`와 다르다. 스트리밍 레벨의 `GetOuter()`는 사용되지 않는 잔재 월드를 가리킬 수 있기 때문. `BeginDestroy()` 중에는 GC 순서 문제로 접근하면 안 된다.
