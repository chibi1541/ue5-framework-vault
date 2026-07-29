---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
tags:
  - World_cpp
---

```cpp
/** returns the AWorldSettings actor associated with this world */
AWorldSettings* GetWorldSettings(bool bCheckStreamingPersistent = false, bool bChecked = true) const
{
    AWorldSettings* WorldSettings = nullptr;
    if (PersistentLevel)
    {
        // haker: do you remember that we create AWorldSettings setting its outer as PersistentLevel?
        WorldSettings = PersistentLevel->GetWorldSettings(bChecked);
        if (bCheckStreamingPersistent)
        {
            //...
        }
    }
    return WorldSettings;
}
```

## 설명
- [[UWorld/UWorld.InitWorld|UWorld::InitWorld]]에서 호출. `PersistentLevel`에 종속되어 스폰된 `AWorldSettings`를 되돌려준다. ([[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]에서 `PersistentLevel->SetWorldSettings(WorldSettings)`로 미리 설정해둔 것)
