---
related:
  - "[[UPlayerInput/UPlayerInput.EvaluateInputDelegates|UPlayerInput::EvaluateInputDelegates]]"
tags:
  - PlayerInput_cpp
---

```cpp
bool UPlayerInput::EvaluateInputComponentDelegates(UInputComponent* const IC, const TArray<TPair<FKey, FKeyState*>>& KeysWithEvents, const float DeltaTime, const bool bGamePaused)
{
	

}
```

## 설명
- [[UPlayerInput/UPlayerInput.EvaluateInputDelegates|EvaluateInputDelegates]]가 스택의 각 `UInputComponent`마다 호출하는 함수. 해당 컴포넌트에 바인딩된 델리게이트 중 이번 프레임의 `KeysWithEvents`와 매칭되는 것을 찾아 실행 목록에 담는다.
- 반환값 `bool`은 **이 컴포넌트가 입력을 소비해 아래 스택으로의 전달을 막는지(block)** 를 의미한다. `true`면 상위 호출자가 스택 순회를 중단한다.
- 본문은 source에 비어 있어 세부 로직은 아직 정리되지 않음.
