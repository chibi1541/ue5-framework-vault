---
related:
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
  - "[[UWorld/UWorld.SetActiveLevelCollection|UWorld::SetActiveLevelCollection]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FLevelCollection/FLevelCollection|FLevelCollection]]"
tags:
  - World_h
---

```cpp
// haker: RAII (Resource Acquisition is initialization) pattern:
// - change ActiveLevelCollection in UWorld
class FScopedLevelCollectionContextSwitch
{
public:
    /**
     * constructor that will save the current relevant values of InWorld
     * and set the collection's context values for InWorld 
     */
    FScopedLevelCollectionContextSwitch(int32 InLevelCollectionIndex, UWorld* const InWorld)
        : World(InWorld)
        , SavedTickingCollectionIndex(InWorld ? InWorld->GetActiveLevelCollectionIndex() : INDEX_NONE)
    {
        if (World)
        {
            World->SetActiveLevelCollection(InLevelCollectionIndex);
        }
    }

    /** the destructor restores the context on the world that was saved in the constructor */
    ~FScopedLevelCollectionContextSwitch()
    {
        if (World)
        {
            World->SetActiveLevelCollection(SavedTickingCollectionIndex);
        }
    }

private:
    class UWorld* World;
    int32 SavedTickingCollectionIndex;
};
```

## 설명
- 이름에 `Scoped`가 들어가면 RAII 패턴이므로 생성자/소멸자만 보면 된다.
- 생성자에서 기존 `ActiveLevelCollectionIndex`를 `SavedTickingCollectionIndex`에 저장하고, [[UWorld/UWorld.SetActiveLevelCollection|UWorld::SetActiveLevelCollection()]]으로 인자로 받은 인덱스의 컬렉션으로 교체한다. 소멸자에서 저장해둔 인덱스로 되돌린다.
- 내부적으로는 `UWorld`의 `CurrentLevel`을 해당 [[FLevelCollection/FLevelCollection|FLevelCollection]]의 `PersistentLevel`로 바꿔놓았다가 스코프를 벗어나면 원래대로 복원하는 효과다.
- [[UWorld/UWorld.Tick|UWorld::Tick]]에서 컬렉션 단위로 tick을 돌 때 사용된다.
