---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[FActorTickFunction/FActorTickFunction|FActorTickFunction]]"
  - "[[FActorThreadContext/FActorThreadContext|FActorThreadContext]]"
  - "[[AActor/AActor.RegisterAllActorTickFunctions|AActor::RegisterAllActorTickFunctions]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[UActorComponent/UActorComponent.RegisterComponentTickFunctions|UActorComponent::RegisterComponentTickFunctions]]"
tags:
  - Actor_cpp
---

```cpp
/** virtual call chain to register all tick functions for the actor class hierarchy */
virtual void RegisterActorTickFunctions(bool bRegister)
{
    if (bRegister)
    {
        if (PrimaryActorTick.bCanEverTick)
        {
            PrimaryActorTick.Target = this;
            PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartsWithTickEnabled || PrimaryActorTick.IsTickFunctionEnabled());
            PrimaryActorTick.RegisterTickFunction(GetLevel());
        }
    }
    else
    {
        if (PrimaryActorTick.IsTickFunctionRegistered())
        {
            PrimaryActorTick.UnRegisterTickFunction();
        }
    }

    // haker: cache the current Actor's TickFunction in TLS
    // - this TLS variable could be used in successive virtual call
    // 영문을 모를 코드 사용...
    FActorThreadContext::Get().TestRegisterTickFunctions = this;
}
```

## 설명
- 액터의 `PrimaryActorTick`([[FActorTickFunction/FActorTickFunction|FActorTickFunction]])을 실제로 등록/해제하는 virtual 함수. 클래스 계층을 따라 override 되며 virtual call chain을 이룬다. [[UActorComponent/UActorComponent.RegisterComponentTickFunctions|UActorComponent::RegisterComponentTickFunctions()]]의 액터 버전에 해당한다.
- 등록 시 `bCanEverTick`이 true인 경우에만 진행한다. `Target`을 자신으로 잡고, [[FTickFunction/FTickFunction.SetTickFunctionEnable|SetTickFunctionEnable()]]로 활성 여부를 정한 뒤 [[FTickFunction/FTickFunction.RegisterTickFunction|RegisterTickFunction(GetLevel())]]로 레벨의 `FTickTaskLevel`에 등록한다.
- 활성 여부는 `bStartsWithTickEnabled || IsTickFunctionEnabled()`로 결정한다. 이미 켜져 있던 상태를 유지하기 위한 or 조건이다.
- 해제(`bRegister == false`) 시에는 [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]로 확인 후 `UnRegisterTickFunction()`을 호출한다.
- 마지막에 [[FActorThreadContext/FActorThreadContext|FActorThreadContext]]의 TLS 변수에 자기 자신을 캐싱한다. 호출자인 [[AActor/AActor.RegisterAllActorTickFunctions|RegisterAllActorTickFunctions()]]가 이 값을 `check()`로 검증한다.
