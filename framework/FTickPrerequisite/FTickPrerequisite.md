---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickFunction/FTickFunction.AddPrerequisite|FTickFunction::AddPrerequisite]]"
  - "[[UObject/UObject|UObject]]"
tags:
  - EngineBaseTypes_h
---

```cpp
/** this is small structure to hold prerequisite tick functions */
struct FTickPrerequisite
{
    // PrerequisiteTickFunction이 유효한지는 PrerequisiteObject(이 경우 UWorld)가 valid한지로 판단
    /** tick functions live inside of UObjects, so we need a separate weak pointer to the UObject solely for the purpose of determining if PrerequisiteTickFunction is still valid */
    // haker: normally FTickFunction's aliveness is determined by UObject(ActorTickFunction -> AActor, ActorComponentTickFunction -> UActorComponent)
    TWeakObjectPtr<UObject> PrerequisiteObject;

    /** pointer to the actual tick function and must be completed prior to our tick running */
    FTickFunction* PrerequisiteTickFunction;

    FTickPrerequisite(UObject* TargetObject, struct FTickFunction& TargetTickFunction)
        : PrerequisiteObject(TargetObject)
        , PrerequisiteTickFunction(&TargetTickFunction)
    {}
};
```

## 설명
- 선행 tick function 하나를 담는 작은 구조체로, [[FTickFunction/FTickFunction|FTickFunction]]의 `Prerequisites` 배열 원소가 된다.
- `PrerequisiteTickFunction`은 자신보다 먼저 완료되어야 하는 tick function의 raw pointer다.
- tick function은 `UObject` 안에 존재하므로 그 자체로는 수명을 알 수 없다. 그래서 소유 [[UObject/UObject|UObject]]를 `TWeakObjectPtr`로 따로 들고, `PrerequisiteObject`가 유효한지로 `PrerequisiteTickFunction`의 유효성을 판별한다.
- 소유자는 보통 `FActorTickFunction` → `AActor`, `FActorComponentTickFunction` → `UActorComponent`이며, physics tick function의 경우 `UWorld`가 된다.
