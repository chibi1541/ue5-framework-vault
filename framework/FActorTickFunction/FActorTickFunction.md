---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]"
  - "[[AActor/AActor|AActor]]"
tags:
  - EngineBaseTypes_h
---

```cpp
USTRUCT()
struct FActorTickFunction : public FTickFunction
{
	/**  AActor  that is the target of this tick **/
#if UE_WITH_REMOTE_OBJECT_HANDLE
		TObjectPtr<AActor> Target;
#else
		class AActor*	Target;
#endif
};

// AActor의 FTickFunction
/**
 * Primary Actor tick function, which calls TickActor().
 * Tick functions can be configured to control whether ticking is enabled, at what time during a frame the update occurs, and to set up tick dependencies.
 * @see <https://docs.unrealengine.com/API/Runtime/Engine/Engine/FTickFunction>
 * @see AddTickPrerequisiteActor(), AddTickPrerequisiteComponent()
 */
UPROPERTY(EditDefaultsOnly, Category=Tick)
struct FActorTickFunction PrimaryActorTick;
```

## 설명
- [[FTickFunction/FTickFunction|FTickFunction]]을 상속한 [[AActor/AActor|AActor]] 전용 tick function으로, `TickActor()`를 호출한다.
- `Target`이 tick 대상인 `AActor`를 가리킨다. [[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]과 구조는 동일하고 `Target` 타입만 다르다.
- `AActor`는 이를 `PrimaryActorTick` 멤버로 갖는다. tick 활성화 여부, 프레임 내 실행 시점, tick 의존성(`AddTickPrerequisiteActor()`, `AddTickPrerequisiteComponent()`)을 설정할 수 있다.
