---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
  - "[[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions]]"
  - "[[FTickFunction/FTickFunction.FInternalData|FTickFunction::FInternalData]]"
  - "[[FPhysicsTickFunction/FPhysicsTickFunction|FPhysicsTickFunction]]"
tags:
  - EngineBaseTypes_h
---

```cpp
/** Determines which ticking group a tick function belongs to. */
UENUM(BlueprintType)
enum ETickingGroup : int
{
	/** Any item that needs to be executed before physics simulation starts. */
	TG_PrePhysics UMETA(DisplayName="Pre Physics"),

	/** Special tick group that starts physics simulation. */							
	TG_StartPhysics UMETA(Hidden, DisplayName="Start Physics"),

	/** Any item that can be run in parallel with our physics simulation work. */
	TG_DuringPhysics UMETA(DisplayName="During Physics"),

	/** Special tick group that ends physics simulation. */
	TG_EndPhysics UMETA(Hidden, DisplayName="End Physics"),

	/** Any item that needs rigid body and cloth simulation to be complete before being executed. */
	TG_PostPhysics UMETA(DisplayName="Post Physics"),

	/** Any item that needs the update work to be done before being ticked. */
	TG_PostUpdateWork UMETA(DisplayName="Post Update Work"),

	/** Catchall for anything demoted to the end. */
	TG_LastDemotable UMETA(Hidden, DisplayName = "Last Demotable"),

	/** Special tick group that is not actually a tick group. After every tick group this is repeatedly re-run until there are no more newly spawned items to run. */
	TG_NewlySpawned UMETA(Hidden, DisplayName="Newly Spawned"),

	TG_MAX,
};
```

## 설명
- [[FTickFunction/FTickFunction|FTickFunction]]이 어느 ticking group에 속하는지를 나타내는 enum. group 순서가 곧 프레임 내 tick 실행 순서를 결정한다.
- physics를 기준으로 구간이 나뉜다: `TG_PrePhysics` → `TG_StartPhysics` → `TG_DuringPhysics` → `TG_EndPhysics` → `TG_PostPhysics` → `TG_PostUpdateWork`.
- `TG_DuringPhysics`는 physics simulation과 병렬로 돌아도 되는 작업용이고, `TG_PostPhysics`는 rigid body/cloth simulation이 끝나야 하는 작업용이다.
- `TG_StartPhysics`/`TG_EndPhysics`는 physics simulation을 시작/종료하는 특수 group으로 `Hidden`이다.
- `TG_LastDemotable`은 맨 뒤로 밀린 것들을 받는 catchall이다.
- `TG_NewlySpawned`는 실제 tick group이 아니다. 매 tick group이 끝난 뒤, 새로 spawn된 항목이 더 없을 때까지 반복 실행된다.
- tick 순서를 나누는 의미와 더불어, task들이 병렬로 실행되는 과정에서 **sync point**(join이 호출되는 지점) 역할도 한다.
- `TG_DuringPhysics` 구간에는 physics thread가 병렬로 돌고 있다. 이때 line trace나 collision query를 하면 lock이 걸려 병목이 생기므로 주의해야 한다.
- `TG_PostUpdateWork`는 물리 시뮬레이션(`TG_EndPhysics`/`TG_PostPhysics`)이 모두 끝난 **이후**에 실행된다. 이번 프레임의 최종 위치·회전이 확정된 뒤에 돌아야 하는 로직용이며, 대표적으로 `CameraComponent`/`SpringArm`이 여기에 속한다. 카메라가 물리 업데이트보다 먼저 tick하면 한 프레임 밀린 것처럼 떨리기 때문이다.
- `TG_LastDemotable`은 더 이상 밀 데가 없어 맨 뒤로 강등(demote)된 것들의 처리 순서다. `Hidden`이라 에디터에서 직접 선택할 수 없고, `AddTickPrerequisiteActor`/`AddTickPrerequisiteComponent` 같은 의존성 체인을 따라가다 마지막 정식 group(`TG_PostUpdateWork`)보다 더 뒤로 밀려야 할 때 엔진 스케줄러가 할당하는 안전장치다.
- `TG_NewlySpawned`는 한 프레임의 정식 group(`TG_PrePhysics` ~ `TG_LastDemotable`)을 다 돌린 뒤, 그 과정에서 새로 spawn된 액터를 처리하기 위한 group이다. 처리 도중 또 새 액터가 spawn될 수 있으므로 더 없을 때까지 반복 실행되며, 목적은 늦게 태어난 액터도 같은 프레임 안에서 최소 한 번은 tick 하도록 보장하는 것이다.
