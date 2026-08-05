---
related:
  - "[[UEnhancedInputWorldSubsystem/UEnhancedInputWorldSubsystem.TickPlayerInput|UEnhancedInputWorldSubsystem::TickPlayerInput]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateKeyMapState|UPlayerInput::EvaluateKeyMapState]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateInputDelegates|UPlayerInput::EvaluateInputDelegates]]"
tags:
  - PlayerInput_cpp
---

```cpp
void UPlayerInput::ProcessInputStack(const TArray<UInputComponent*>& InputComponentStack, const float DeltaTime, const bool bGamePaused)
{
	static TArray<TPair<FKey, FKeyState*>> KeysWithEvents;

// Evaluate the current state of the key map. This will take the event accumulators and put them into the actual values of the key state map.
	EvaluateKeyMapState(DeltaTime, bGamePaused, OUT KeysWithEvents);
	
	// Determine potential delegates
	EvaluateInputDelegates(InputComponentStack, DeltaTime, bGamePaused, KeysWithEvents);
}

```

## 설명
- tick당 입력 처리를 두 단계로 나누는 함수.
  1. [[UPlayerInput/UPlayerInput.EvaluateKeyMapState|EvaluateKeyMapState]] — 누적해 둔 accumulator를 `KeyStateMap`의 **확정값**으로 옮기고, 이번 프레임에 이벤트가 있는 키만 `KeysWithEvents`로 추려낸다.
  2. [[UPlayerInput/UPlayerInput.EvaluateInputDelegates|EvaluateInputDelegates]] — 그 결과를 가지고 `InputComponentStack`을 훑으며 실행할 델리게이트를 결정한다.
- 상태 확정과 델리게이트 실행을 분리하는 이유는, 델리게이트가 실행되는 동안 키 상태가 계속 바뀌면 같은 프레임 안에서 컴포넌트마다 다른 상태를 보게 되기 때문이다. 먼저 전부 확정한 뒤 소비한다.
- `KeysWithEvents`가 `static`인 것도 매 프레임 할당을 피하기 위한 최적화다. 전체 `KeyStateMap`이 아니라 **이벤트가 있는 키만** 넘겨 다음 단계의 순회 비용을 줄인다.
