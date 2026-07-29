---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
tags:
  - World_h
---

```cpp
/** returns the default physics volume and creates it if necessary */
APhysicsVolume* GetDefaultPhysicsVolume() const { return DefaultPhysicsVolume ? ToRawPtr(DefaultPhysicsVolume) : InternalGetDefaultPhysicsVolume(); }

APhysicsVolume* InternalGetDefaultPhysicsVolume() const
{
    // haker: if we don't have any DefaultPhysicsVolume yet, create new one
    if (DefaultPhysicsVolume == nullptr)
    {
        // haker: we get the PhysicssVolume's class to instantiate:
        // - WorldSettings have class definitions which are needed for the world, which could change overall behavior for world
        // - you could also override the class in WorldSettings
        AWorldSettings* WorldSettings = GetWorldSettings(false, false);
        UClass* DefaultPhysicsVolumeClass = (WorldSettings ? WorldSettings->DefaultPhysicsVolumeClass : nullptr);

        if (DefaultPhysicsVolumeClass == nullptr)
        {
            DefaultPhysicsVolumeClass = ADefaultPhysicsVolume::StaticClass();
        }

        // spawn volume:
        // haker: here is FActorSpawnParameters
        FActorSpawnParameters SpawnParams;
        SpawnParams.bAllowDuringConstructionScript = true;
        {
            UWorld* MutableThis = const_cast<UWorld*>(this);
            MutableThis->DefaultPhysicsVolume = MutableThis->SpawnActor<APhysicsVolume>(DefaultPhysicsVolumeClass, SpawnParams);
            MutableThis->DefaultPhysicsVolume->Priority = -1000000;
        }
    }
    return DefaultPhysicsVolume;
}
```

## 설명
- [[UWorld/UWorld.InitWorld|UWorld::InitWorld]]에서 호출. `DefaultPhysicsVolume`이 이미 있으면 그대로 반환하고, 없으면 `InternalGetDefaultPhysicsVolume`에서 생성한다.
- 생성 시 [[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]로 `SpawnActor`를 호출한다. Physics Scene이 없어도 콜리전 판정을 위해 PhysicsVolume 자체는 생성해야 한다.
