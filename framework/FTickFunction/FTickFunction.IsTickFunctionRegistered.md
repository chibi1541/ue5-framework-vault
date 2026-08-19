---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[AActor/AActor.RegisterActorTickFunctions|AActor::RegisterActorTickFunctions]]"
  - "[[FTickFunction/FTickFunction.AddPrerequisite|FTickFunction::AddPrerequisite]]"
  - "[[FTickFunction/FTickFunction.FInternalData|FTickFunction::FInternalData]]"
  - "[[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions]]"
tags:
  - EngineBaseTypes_h
---

```cpp
bool IsTickFunctionRegistered() const 
{
	return (InternalData && InternalData->bRegisterd);
}
```

## 설명
- tick function이 등록되었는지를 판별한다.
- `InternalData`는 등록된 tick function에만 lazy 할당되는 hot/cold 분리 데이터이므로, `InternalData`의 존재 자체가 1차 조건이 된다.
- 실제 등록 완료 여부는 [[FTickFunction/FTickFunction.RegisterTickFunction|RegisterTickFunction()]]이 마지막에 세우는 `InternalData->bRegistered`로 판별한다.
