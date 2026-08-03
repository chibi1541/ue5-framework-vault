---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]"
  - "[[AActor/AActor.RegisterActorTickFunctions|AActor::RegisterActorTickFunctions]]"
  - "[[FActorThreadContext/FActorThreadContext|FActorThreadContext]]"
  - "[[UActorComponent/UActorComponent.RegisterAllComponentTickFunctions|UActorComponent::RegisterAllComponentTickFunctions]]"
  - "[[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|UObjectBaseUtility::IsTemplate]]"
tags:
  - Actor_cpp
---

```cpp
/** when called, will call the virtual call chain to register all of the tick functions for both the actor and optionally all components */
void RegisterAllActorTickFunctions(bool bRegister, bool bDoComponents)
{
    // CDO 혹은 ArcheType
    if (!IsTemplate())
    {
        // prevent repeated redundant attempts
        if (bTickFunctionsRegistered != bRegister)
        {
            FActorThreadContext& ThreadContext = FActorThreadContext::Get();

            RegisterActorTickFunctions(bRegister);
            bTickFunctionsRegistered = bRegister;

            // haker: validate TestRegisterTickFunctions updated in RegisterActorTickCuntions, then reset it
            // 왜 굳이 이 단계를 TLS 변수를 써가면서 체크하지? 어차피 게임 쓰레드에서만 접근가능하지 않나?
            check(ThreadContext.TestRegisterTickFunctions == this);
            ThreadContext.TestRegisterTickFunctions = nullptr;
        }

        // haker: remember we set bDoComponents as false in our previous callstack
        if (bDoComponents)
        {
            for (UActorComponent* Component : GetComponents())
            {
                if (Component)
                {
                    Component->RegisterAllComponentTickFunctions(bRegister);
                }
            }
        }

        if (bAsyncPhysicsTickEnabled)
        {
            //...
        }

        // search (goto 074)
    }
}
```

## 설명
- 액터 자신의 tick function과, 선택적으로(`bDoComponents`) 소유 컴포넌트들의 tick function까지 등록하는 진입점이다. [[UActorComponent/UActorComponent.RegisterAllComponentTickFunctions|UActorComponent::RegisterAllComponentTickFunctions()]]와 같은 역할을 액터 쪽에서 하는 함수.
- [[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|IsTemplate()]]로 CDO/archetype을 걸러낸다. 템플릿 객체는 실제로 tick 되지 않으므로 등록 대상이 아니다.
- `bTickFunctionsRegistered != bRegister` 조건으로 같은 등록/해제를 중복 수행하지 않게 막는다. 실제 등록은 virtual인 [[AActor/AActor.RegisterActorTickFunctions|RegisterActorTickFunctions()]]에 위임한다 — 상속 계층을 따라 virtual call chain으로 내려간다.
- `RegisterActorTickFunctions()`가 [[FActorThreadContext/FActorThreadContext|FActorThreadContext]]의 TLS 변수 `TestRegisterTickFunctions`에 자기 자신을 써두고, 여기서 `check()`로 그 값이 `this`인지 검증한 뒤 다시 `nullptr`로 리셋한다. 즉 virtual chain의 최상위 구현까지 제대로 호출되었는지를 확인하는 장치다.
- [[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents()]]에서 넘어올 때는 `bDoComponents`가 false이므로, 컴포넌트 tick 등록 루프는 타지 않는다. 컴포넌트 쪽은 각 컴포넌트가 register 될 때 따로 처리된다.
