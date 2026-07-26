---
related:
  - "[[UWorld/UWorld.CreateWorld|UWorld::CreateWorld]]"
  - "[[UWorld/UWorld|UWorld]]"
tags:
  - WorldInitializationValues_h
---

```cpp
/** struct containing a collection of optional parameters for initialization of a world */
// haker: think of this pattern as one struct encapsulating multiple parameters for the code readability
// - this struct contains all necessary options to create world
struct FWorldInitializationValues
{
    /** should the scenes (physics, rendering) be created */
    // haker: whether we create worlds (render world, physics world, ...)
    // Scene은 FScene을 의미
    uint32 bInitializeScenes:1;

    /** Should the physics scene be created. bInitializeScenes must be true for this to be considered. */
    uint32 bCreatePhysicsScene:1;

    /** Are collision trace calls valid within this world. */
    uint32 bEnableTraceCollision:1;

    //...
};
```

## 설명
- 월드를 만들 때 필요한 옵션들을 하나의 struct로 묶은 패턴. 인자를 길게 나열하는 대신 가독성을 확보한다.
- `bInitializeScenes`는 render world(`FScene`), physics world 같은 scene들을 생성할지를 결정한다. `bCreatePhysicsScene`은 `bInitializeScenes`가 true일 때만 의미가 있다.
- 모두 `uint32 :1` 비트 필드로 선언되어 플래그 모음처럼 동작한다.
