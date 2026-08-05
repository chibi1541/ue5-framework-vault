---
related:
  - "[[UPlayerInput/UPlayerInput.ProcessInputStack|UPlayerInput::ProcessInputStack]]"
  - "[[UEnhancedPlayerInput/UEnhancedPlayerInput.InputKey|UEnhancedPlayerInput::InputKey]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
tags:
  - EnhancedInputWorldSubsystem_cpp
---

```cpp
void UEnhancedInputWorldSubsystem::TickPlayerInput(float DeltaTime)
{
	static TArray<UInputComponent*> InputStack;
	InputStack.Reset();

	// Build the input stack
	{
		for (int32 i = 0; i < CurrentInputStack.Num(); ++i)
		{
			if (UInputComponent* IC = CurrentInputStack[i].Get())
			{
				InputStack.Push(IC);
			}
			else
			{
				CurrentInputStack.RemoveAt(i--);
			}
		}
	}
	
	// Process input stack on the player input
	PlayerInput->Tick(DeltaTime);
	PlayerInput->ProcessInputStack(InputStack, DeltaTime, GetWorld()->IsPaused());
}
```

## 설명
- **이번 프레임에 쌓인 input 이벤트를 tick에서 한 번에 처리하는** 진입점. 입력 수집([[UEnhancedPlayerInput/UEnhancedPlayerInput.InputKey|InputKey]] 계열)과 입력 소비(tick)가 분리되어 있는 구조의 후반부다.
- `CurrentInputStack`은 weak ptr 배열이라 이미 파괴된 `UInputComponent`가 섞여 있을 수 있다. 순회하면서 유효한 것만 `InputStack`에 push하고, 무효한 항목은 `RemoveAt(i--)`로 제거한다(제거 후 인덱스 보정).
- `InputStack`이 `static`인 것은 매 프레임 배열을 새로 할당하지 않기 위한 최적화다. 그래서 시작할 때 `Reset()`을 먼저 호출한다.
- 최종적으로 [[UPlayerInput/UPlayerInput.ProcessInputStack|ProcessInputStack]]에 스택과 `bGamePaused`(`GetWorld()->IsPaused()`)를 넘긴다.
