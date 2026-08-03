---
related:
  - "[[UActorComponent/UActorComponent.RegisterAllComponentTickFunctions|UActorComponent::RegisterAllComponentTickFunctions]]"
  - "[[UActorComponent/UActorComponent.SetupActorComponentTickFunction|UActorComponent::SetupActorComponentTickFunction]]"
  - "[[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[AActor/AActor.RegisterActorTickFunctions|AActor::RegisterActorTickFunctions]]"
tags:
  - ActorComponent_cpp
---

```cpp
virtual void RegisterComponentTickFunctions(bool bRegister)
{
    if (bRegister)
    {
        if (SetupActorComponentTickFunction(&PrimaryComponentTick))
        {
            PrimaryComponentTick.Target = this;
        }
    }
    else
    {
        if (PrimaryComponentTick.IsTickFunctionRegistered())
        {
            PrimaryComponentTick.UnRegisterTickFunction();
        }
    }
}
```

## 설명
- `virtual` 함수이므로 사용자가 오버라이딩해 자신만의 tick function 등록을 추가할 수 있다.
- 등록 시: [[UActorComponent/UActorComponent.SetupActorComponentTickFunction|SetupActorComponentTickFunction()]]에 자신의 [[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]인 `PrimaryComponentTick`을 넘기고, 성공하면 `Target`을 자기 자신으로 설정한다. tick이 실행될 때 이 `Target`이 tick 대상이 된다.
- 해제 시: [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]로 등록 여부를 확인한 뒤에만 `UnRegisterTickFunction()`을 호출한다.
