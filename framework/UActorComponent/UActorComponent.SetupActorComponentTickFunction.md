---
related:
  - "[[UActorComponent/UActorComponent.RegisterComponentTickFunctions|UActorComponent::RegisterComponentTickFunctions]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|UObjectBaseUtility::IsTemplate]]"
  - "[[ULevel/ULevel|ULevel]]"
tags:
  - ActorComponent_cpp
---

```cpp
bool SetupActorComponentTickFunction(FTickFunction* TickFunction)
{
    if (TickFunction->bCanEverTick && !IsTemplate())
    {
        AActor* MyOwner = GetOwner();
        if (!MyOwner || !MyOwner->IsTemplate())
        {
            // ActorComponent는 월드는 캐싱하지만 Level은 직접 캐싱하지 않음
            ULevel* ComponentLevel = (MyOwner ? MyOwner->GetLevel() : ToRawPtr(GetWorld()->PersistentLevel));
            // haker: if TickFunction is not registered, just TickState will be updated
            // - we have seen SetTickFunctionEnable(), but let's see one more again briefly
            // 여기서 TickFunction을 FTickTaskLevel에 등록
            TickFunction->SetTickFunctionEnable(TickFunction->bStartsWithTickEnabled || TickFunction->IsTickFunctionEnabled());

            // haker: we need ULevel where the component resides in, cuz FTickFunction is resides in TickTaskLevel!
            TickFunction->RegisterTickFunction(ComponentLevel);
            return true;
        }
    }
    return false;

    // haker: SetupActorComponentTickFunction makes sure that TickFunction is registered
}
```

## 설명
- `FTickFunction`을 인자로 받아 tick을 enable/disable 하면서 등록까지 마치는 함수. tick function이 등록되었음을 보장한다.
- `bCanEverTick`이 false거나 자신 혹은 owner가 [[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|IsTemplate()]]이면 false를 반환하고 아무 것도 하지 않는다.
- `UActorComponent`는 world는 캐싱하지만 [[ULevel/ULevel|ULevel]]은 직접 캐싱하지 않는다. 그래서 owner가 있으면 `MyOwner->GetLevel()`, 없으면 `GetWorld()->PersistentLevel`로 level을 구한다. `FTickFunction`은 level 단위의 `FTickTaskLevel`에 들어가므로 level이 반드시 필요하다.
- 앞서 본 [[FTickFunction/FTickFunction.SetTickFunctionEnable|SetTickFunctionEnable()]]과 [[FTickFunction/FTickFunction.RegisterTickFunction|RegisterTickFunction(ULevel*)]]을 함께 호출한다. 아직 등록 전이므로 `SetTickFunctionEnable()`은 `TickState`만 갱신하고, 실제 큐 삽입은 뒤이은 `RegisterTickFunction()`이 처리한다.
- enable 여부는 `bStartsWithTickEnabled || IsTickFunctionEnabled()`로 결정한다.
