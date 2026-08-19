---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions]]"
  - "[[Enum/ETickingGroup|ETickingGroup]]"
tags:
  - World_h
---

```cpp
/** tick function that starts the physics tick */
struct FStartPhysicsTickFunction : public FTickFunction
{
    /** world this tick function belongs to */
    UWorld* Target;
};

/** tick function that ends the physics tick */
struct FEndPhysicsTickFunction : public FTickFunction
{
    /** world this tick function belongs to */
    UWorld* Target;
};
```

## 설명
- physics simulation의 시작/종료를 담당하는 tick function으로, 둘 다 [[FTickFunction/FTickFunction|FTickFunction]]을 상속한다.
- `Target`은 이 tick function이 속한 [[UWorld/UWorld|UWorld]]를 가리킨다. `FActorTickFunction`이 `AActor`를, `FActorComponentTickFunction`이 `UActorComponent`를 가리키는 것과 같은 구조다.
- [[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions()]]에서 각각 `TG_StartPhysics` / `TG_EndPhysics` group으로 등록된다([[Enum/ETickingGroup|ETickingGroup]]).
