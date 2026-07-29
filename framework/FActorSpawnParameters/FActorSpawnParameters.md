---
related:
  - "[[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]"
  - "[[Enum/ESpawnActorCollisionHandlingMethod|ESpawnActorCollisionHandlingMethod]]"
  - "[[UWorld/UWorld.GetDefaultPhysicsVolume|UWorld::GetDefaultPhysicsVolume]]"
tags:
  - World_h
---

```cpp
/** struct of optional parameters passed to SpawnActor function(s) */
// haker: I reduce its member variables drastically
struct FActorSpawnParameters
{
		// FName 유니크한 값 -> 내부는 NameEntryIndex = WorldSettings, Number = 0의 형태로 나누어서 보관
		// NameEntryIndex를 별도의 메모리로 항상 올려놓고 중복을 체크함
    /** a name to assign as the Name of the Actor being spawned; if no value is specified, the name of the spawned Actor will be automatically generated using the form [Class]_[Number] */
    // haker: default format of FName is [ClassName]_[Number]:
    // - e.g. WorldSettings_0
    FName Name;

    /** method for resolving collisions at the spawn point; undefined means no override, use the actor's setting */
    ESpawnActorCollisionHandlingMethod SpawnCollisionHandlingOverride;

    /** determines whether or not the actor may be spawned when running a construction script; if true spawning will fail if a construction script is being run */
    uint8 bAllowDuringConstructionScript : 1;
};

/** defines available strategies for handling the case where an actor is spawned in such a way that it penetrates blocking collisions */
// haker: spawning policy depending on collision between object and obstacles
enum class ESpawnActorCollisionHandlingMethod : uint8
{
    /** fall back to default settings */
    Undefined,
    // 충돌 신경 쓰지 말고 무조건 스폰
    /** actor will spawn in desired location, regardless of collisions */
    AlawysSpawn,
    // 스폰 시에 이미 충돌할 것 같은 물체가 있는 경우 스폰하지 않음
    /** actor will try to find a nearby non-colliding location (based on shape components) but will always spawn even if one cannot be found */
    // haker: spawn but adjust a little bit and spawn non-collising location
    AdjustIfPossibleButAlwaysSpawn,
    //...
};
```

## 설명
- `SpawnActor` 계열 함수에 전달하는 옵션 struct. 주요하게 볼 건 `Name`(지정하지 않으면 `[Class]_[Number]` 형태로 자동 생성)과 `SpawnCollisionHandlingOverride`.
- `SpawnCollisionHandlingOverride`는 [[Enum/ESpawnActorCollisionHandlingMethod|ESpawnActorCollisionHandlingMethod]]로 스폰 시 충돌 처리 방식을 결정한다.
