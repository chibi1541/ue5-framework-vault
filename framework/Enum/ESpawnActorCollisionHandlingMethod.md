---
related:
  - "[[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]"
tags:
  - EngineTypes_h
---

```cpp
/** Defines available strategies for handling the case where an actor is spawned in such a way that it penetrates blocking collision. */
UENUM(BlueprintType)
enum class ESpawnActorCollisionHandlingMethod : uint8
{
	/** Fall back to default settings. */
	Undefined								UMETA(DisplayName = "Default"),
	/** Actor will spawn in desired location, regardless of collisions. */
	AlwaysSpawn								UMETA(DisplayName = "Always Spawn, Ignore Collisions"),
	/** Actor will try to find a nearby non-colliding location (based on shape components), but will always spawn even if one cannot be found. */
	AdjustIfPossibleButAlwaysSpawn			UMETA(DisplayName = "Try To Adjust Location, But Always Spawn"),
	/** Actor will try to find a nearby non-colliding location (based on shape components), but will NOT spawn unless one is found. */
	AdjustIfPossibleButDontSpawnIfColliding	UMETA(DisplayName = "Try To Adjust Location, Don't Spawn If Still Colliding"),
	/** Actor will fail to spawn. */
	DontSpawnIfColliding					UMETA(DisplayName = "Do Not Spawn"),
};
```

## 설명
- 액터 스폰 시 충돌 처리 전략. [[FActorSpawnParameters/FActorSpawnParameters|FActorSpawnParameters]]의 `SpawnCollisionHandlingOverride`에서 사용하는 실제(전체) 정의.
- `Undefined`(기본값), `AlwaysSpawn`(충돌 무시), `AdjustIfPossibleBut...`(위치 조정 시도) 계열, `DontSpawnIfColliding`(충돌 시 스폰 실패)으로 구성된다.
