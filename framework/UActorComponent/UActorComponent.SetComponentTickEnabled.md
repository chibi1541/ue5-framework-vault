---
related:
  - "[[UActorComponent/UActorComponent.Activate|UActorComponent::Activate]]"
  - "[[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]"
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|UObjectBaseUtility::IsTemplate]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
tags:
  - ActorComponent_cpp
---

```cpp
void UActorComponent::SetComponentTickEnabled(bool bEnabled)
{
	if (PrimaryComponentTick.bCanEverTick && !IsTemplate())
	{
		PrimaryComponentTick.SetTickFunctionEnable(bEnabled);
	}
}
```

## 설명
- 컴포넌트의 tick on/off를 제어한다. 실제로는 자신의 [[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]인 `PrimaryComponentTick`에 위임한다.
- `bCanEverTick`이 false면 애초에 tick function이 등록될 수 없으므로 아무 것도 하지 않는다. 이 값은 default에서만 설정 가능하다.
- `IsTemplate()`(CDO/archetype)인 경우도 제외한다. 템플릿은 실제로 tick 하는 인스턴스가 아니기 때문.
