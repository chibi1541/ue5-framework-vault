---
related:
  - "[[UPlayerInput/UPlayerInput.ProcessInputStack|UPlayerInput::ProcessInputStack]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateKeyMapState|UPlayerInput::EvaluateKeyMapState]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateInputComponentDelegates|UPlayerInput::EvaluateInputComponentDelegates]]"
tags:
  - PlayerInput_cpp
---

```cpp
void UPlayerInput::EvaluateInputDelegates(const TArray<UInputComponent*>& InputComponentStack, const float DeltaTime, const bool bGamePaused, const TArray<TPair<FKey, FKeyState*>>& KeysWithEvents)
{
	using namespace UE::Input;

	PrepareInputDelegatesForEvaluation(InputComponentStack, DeltaTime, bGamePaused, KeysWithEvents);
	
	// must be called non-recursively and on the game thread
	check(IsInGameThread() && !AxisDelegates.Num() && !VectorAxisDelegates.Num() && !NonAxisDelegates.Num() && !EventIndices.Num());

	int32 StackIndex = InputComponentStack.Num() - 1;

	// Walk the stack, top to bottom
	for ( ; StackIndex >= 0; --StackIndex)
	{
		UInputComponent* const IC = InputComponentStack[StackIndex];
		if (IsValid(IC))
		{
			const bool bDoesComponentBlockFurtherInput = EvaluateInputComponentDelegates(IC, KeysWithEvents, DeltaTime, bGamePaused);
			if (bDoesComponentBlockFurtherInput)
			{
				// stop traversing the stack, all input has been consumed by this InputComponent
                --StackIndex;
				break;
			}
		}
	}
	
	// If the stack index is >=0, then an input component has "blocked" the input.
	// Set any remaining component delegate values to zero. Otherwise, these delegate values
	// have already been set and processed.
	for ( ; StackIndex >= 0; --StackIndex)
	{
		if (UInputComponent* IC = InputComponentStack[StackIndex])
		{
			EvaluateBlockedInputComponent(IC);
		}
	}
	
	SortAndExecuteDelegates();
}
```

## 설명
- `InputComponentStack`을 **위(top, 마지막 인덱스)에서 아래로** 순회한다. 나중에 push된 InputComponent가 더 높은 우선순위를 갖는 스택 구조다.
- 각 컴포넌트에서 [[UPlayerInput/UPlayerInput.EvaluateInputComponentDelegates|EvaluateInputComponentDelegates]]가 `true`(= block)를 반환하면 순회를 멈춘다. 그 아래 컴포넌트들은 입력을 받지 못한다. UI가 열렸을 때 게임 입력이 막히는 동작이 이 구조로 구현된다.
- 두 번째 루프가 중요하다. 막힌(blocked) 나머지 컴포넌트들을 그냥 무시하지 않고 `EvaluateBlockedInputComponent()`로 델리게이트 값을 **0으로 세팅**해 준다. 그렇지 않으면 이동 중에 UI가 열렸을 때 마지막 axis 값이 남아 캐릭터가 계속 움직이는 문제가 생긴다.
- `check(IsInGameThread() && ...)`로 게임 스레드에서 비재귀적으로만 호출되어야 함을 강제한다. 델리게이트 임시 배열들(`AxisDelegates` 등)이 멤버로 재사용되기 때문에 재진입하면 상태가 깨진다.
- 마지막 `SortAndExecuteDelegates()`에서 수집된 델리게이트를 정렬 후 실행한다. 순회 중에 즉시 실행하지 않고 모아 두었다가 한 번에 실행하는 이유는, 실행 순서를 우선순위대로 보장하기 위해서다.
