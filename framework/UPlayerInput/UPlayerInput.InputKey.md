---
related:
  - "[[UEnhancedPlayerInput/UEnhancedPlayerInput.InputKey|UEnhancedPlayerInput::InputKey]]"
  - "[[APlayerController/APlayerController.InputKey|APlayerController::InputKey]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateKeyMapState|UPlayerInput::EvaluateKeyMapState]]"
tags:
  - PlayerInput_cpp
---

```cpp
bool UPlayerInput::InputKey(const FInputKeyEventArgs& Params)
{

// Non-analog key
	else
	{
		FKeyState* ExistingKeyState = KeyStateMap.Find(Params.Key);

		// 여기서 KeyStateMap을 채움
		// first event associated with this key, add it to the map
		FKeyState& KeyState = !ExistingKeyState ? KeyStateMap.Add(Params.Key) : *ExistingKeyState;

		UWorld* World = GetWorld();
		check(World);

		const bool bIsFirstEventForKey = (ExistingKeyState == nullptr) || KeyState.bWasJustFlushed;

		const float WorldRealTimeSeconds = World->GetRealTimeSeconds();

		// If this is the first key press for us and it is a repeat, then we have missed the initial IE_Pressed event.
		// This can happen if you are holding down a key between level transitions and the player controller gets recreated,
		// which means that we will be using a new UPlayerInput object and the KeyState map is emptied.
		// This can ALSO happen if "FlushPressedKeys" gets called, like when we change player controller input modes, and you keep holding down the key 
		// in between those transitions
		if (UE::Input::bAutoReconcilePressedEventsOnFirstRepeat && bIsFirstEventForKey && Params.Event == IE_Repeat && KeyState.EventAccumulator[IE_Pressed].IsEmpty())
		{
			// Mark as having received the IE_Pressed event already, so that we can correctly evaluate the 
			// state of the IE_Repeat event. It is impossible to get a IE_Repeat with an initial IE_Pressed somewhere
			//
			// Without this, the key will incorrectly evaluate as being released/pressed
			// every frame, even if you are just holding it
			KeyState.RawValueAccumulator.X = Params.AmountDepressed;
			KeyState.EventAccumulator[IE_Pressed].Add(++EventCount);
			KeyState.LastUpDownTransitionTime = WorldRealTimeSeconds;
			KeyState.SampleCountAccumulator++;

			// We can assume that if we are getting a "Repeat event" that this key was already down the previous frame and it is down this frame
			KeyState.bDown = true;
			KeyState.bDownPrevious = true;
		}

		switch(Params.Event)
		{
		case IE_Pressed:
		case IE_Repeat:
			KeyState.RawValueAccumulator.X = Params.AmountDepressed;
			KeyState.EventAccumulator[Params.Event].Add(++EventCount);
			if (KeyState.bDownPrevious == false)
			{
				// check for doubleclick
				// note, a tripleclick will currently count as a 2nd double click.
				if ((WorldRealTimeSeconds - KeyState.LastUpDownTransitionTime) < GetDefault<UInputSettings>()->DoubleClickTime)
				{
					KeyState.EventAccumulator[IE_DoubleClick].Add(++EventCount);
				}

				// just went down
				KeyState.LastUpDownTransitionTime = WorldRealTimeSeconds;
			}
			break;
		case IE_Released:
			KeyState.RawValueAccumulator.X = 0.f;
			KeyState.EventAccumulator[IE_Released].Add(++EventCount);
			break;
		case IE_DoubleClick:
			KeyState.RawValueAccumulator.X = Params.AmountDepressed;
			KeyState.EventAccumulator[IE_Pressed].Add(++EventCount);
			KeyState.EventAccumulator[IE_DoubleClick].Add(++EventCount);
			break;
		}
		KeyState.SampleCountAccumulator++;

		// We have now processed this key's state, so we can clear its "just flushed" flag and treat it normally
		KeyState.bWasJustFlushed = false;

	#if !UE_BUILD_SHIPPING
		CurrentEvent		= Params.Event;

		const FString Command = GetBind(Params.Key);
		if(Command.Len())
		{
			return ExecInputCommands(World, *Command,*GLog);
		}
	#endif

		if(Params.Event == IE_Pressed)
		{
			return IsKeyHandledByAction( Params.Key);
		}

		return true;
	}
}
```

## 설명
- 입력 전달 체인의 **종착점**. 여기서 `KeyStateMap`(`FKey` → `FKeyState`)을 채운다. 이후의 실제 델리게이트 실행은 tick에서 [[UPlayerInput/UPlayerInput.EvaluateKeyMapState|EvaluateKeyMapState]]가 이 맵을 읽어서 처리한다.
- 이름에 `Accumulator`가 붙은 이유가 중요하다. 입력 이벤트는 프레임 중 여러 번 도착할 수 있으므로 바로 확정값에 쓰지 않고 `EventAccumulator` / `RawValueAccumulator` / `SampleCountAccumulator`에 **누적**만 해 둔다. tick 때 한 번에 확정값으로 옮긴다.
- `bAutoReconcilePressedEventsOnFirstRepeat` 블록: 첫 이벤트가 `IE_Repeat`로 들어왔다는 것은 `IE_Pressed`를 놓쳤다는 뜻이다. 레벨 전환으로 PlayerController가 재생성되어 `KeyStateMap`이 비었거나, `FlushPressedKeys()` 이후에 키를 계속 누르고 있던 경우에 발생한다. 이때 `IE_Pressed`를 인위적으로 채워 넣지 않으면 키를 누르고만 있어도 매 프레임 pressed/released가 반복되는 것으로 잘못 평가된다.
- 더블클릭 판정: `bDownPrevious`가 false인(= 방금 눌린) 시점에 직전 up/down 전환 시각과의 차이가 `DoubleClickTime` 미만이면 `IE_DoubleClick`을 누적한다. 주석대로 트리플 클릭은 두 번째 더블클릭으로 잡힌다.
- shipping 빌드가 아니면 `GetBind()`로 콘솔 커맨드 바인딩이 있는지 먼저 확인하고, 있으면 `ExecInputCommands()`로 실행한 뒤 반환한다.
- `IE_Pressed`의 반환값은 `IsKeyHandledByAction()`이다. 즉 해당 키가 실제 액션에 매핑되어 있을 때만 "처리됨"으로 보고해, 매핑 없는 키는 상위로 다시 흘려보낸다.
