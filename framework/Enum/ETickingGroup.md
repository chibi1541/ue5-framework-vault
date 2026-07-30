---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
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
