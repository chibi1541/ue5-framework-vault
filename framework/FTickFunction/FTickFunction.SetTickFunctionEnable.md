---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[FTickTaskLevel/FTickTaskLevel.RemoveTickFunction|FTickTaskLevel::RemoveTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction]]"
  - "[[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]"
  - "[[UActorComponent/UActorComponent.SetupActorComponentTickFunction|UActorComponent::SetupActorComponentTickFunction]]"
  - "[[AActor/AActor.RegisterActorTickFunctions|AActor::RegisterActorTickFunctions]]"
tags:
  - TickTaskManager_cpp
---

```cpp
void SetTickFunctionEnable(bool bInEnabled)
{
    // haker: this function is to enable self(tick-function) to tick()
    if (IsTickFunctionRegistered())
    {
        // haker: carefully read the condition in the if statement
        // - state is changed from disabled -> enabled
        if (bInEnabled == (TickState == ETickState::Disabled))
        {
            // haker: InternalData has tick-task-level which it is resides in
            FTickTaskLevel* TickTaskLevel = InternalData->TickTaskLevel;

            // 무조건 remove를 먼저 한번 함
            TickTaskLevel->RemoveTickFunction(this);
            TickState = (bInEnabled ? ETickState::Enabled : ETickState::Disabled);

            TickTaskLevel->AddTickFunction(this);

            // haker: from here, you should see simple-rule in coding:
            // when Enable/Disable FTickFunction, just ***re-insert it***
            // - it doesn't add any complex conditions to handle each case
        }

        if (TickState == ETickState::Disabled)
        {
            InternalData->LastTickGameTimeSeconds = -1.f;
        }
    }
    else
    {
        // haker: if it is NOT registered yet, just update the TickState
        // - when TickFunction is registered, it will handle it as we covered above
        TickState = (bInEnabled ? ETickState::Enabled : ETickState::Disabled);
    }
}
```

## 설명
- tick function 자기 자신의 tick을 활성화/비활성화하는 함수.
- [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]로 등록 여부를 먼저 확인한다. 아직 등록되지 않았다면 `TickState`만 갱신하고 끝낸다. 등록 시점에 위 로직이 다시 처리해 주기 때문이다.
- 등록되어 있다면 `bInEnabled == (TickState == ETickState::Disabled)` 조건, 즉 **상태가 실제로 바뀌는 경우에만** 작업한다. 같은 처리를 두 번 반복하지 않기 위해 언리얼은 이렇게 클래스 내부 상태 플래그를 확인하며 로직을 수행하는 패턴을 자주 쓴다.
- 상태 변경은 항상 `RemoveTickFunction()` → `TickState` 갱신 → `AddTickFunction()`의 **재삽입(re-insert)** 으로 처리한다. 케이스별 분기를 늘리지 않는 단순한 규칙이다.
- `Disabled`가 된 경우 `InternalData->LastTickGameTimeSeconds`를 `-1.f`로 되돌린다.
