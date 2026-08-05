---
related:
  - "[[APlayerController/APlayerController.InputKey|APlayerController::InputKey]]"
  - "[[UPlayerInput/UPlayerInput.InputKey|UPlayerInput::InputKey]]"
  - "[[UEnhancedInputWorldSubsystem/UEnhancedInputWorldSubsystem.TickPlayerInput|UEnhancedInputWorldSubsystem::TickPlayerInput]]"
tags:
  - EnhancedPlayerInput_cpp
---

```cpp
bool UEnhancedPlayerInput::InputKey(const FInputKeyEventArgs& Params)
{
	const bool bResult = Super::InputKey(Params);

	if (Params.Event == IE_Pressed)
	{
		if (Params.Key.IsButtonAxis())
		{
			KeysPressedThisTick.FindOrAdd(Params.Key, FVector(Params.AmountDepressed, 0.0, 0.0));
		}
		else if (UE::Input::bDetectSubTickTapsForNonAxisInputs && Params.Key.IsDigital())
		{
			KeysPressedThisTick.FindOrAdd(Params.Key, FVector(1.0f, 0.0, 0.0));
		}
	}
	else if (Params.Event == IE_Released && UE::Input::bIgnoreHeldDigitalActionKeysOnFlush)
	{
		// Clear bShouldBeIgnored as soon as the release event arrives instead of waiting one full
		// tick (the lifecycle check at the top of EvaluateInputDelegates only clears it after the
		// next tick observes bDown=false && bWasJustFlushed=false). That one-tick lag is the window
		// where a fast re-press lands in the "ignored" state and is silently consumed — the
		// "must press twice to toggle" symptom users see when a UI mapping rebuild flushes the key
		// during the same press that opened the UI.
		for (FEnhancedActionKeyMapping& Mapping : EnhancedActionMappings)
		{
			if (Mapping.bShouldBeIgnored && Mapping.Key == Params.Key)
			{
				Mapping.bShouldBeIgnored = false;
				UE_LOGF(LogEnhancedInput, Verbose, "Key %ls is no longer ignored", *Mapping.Key.ToString());
			}
		}
	}

	return bResult;
}

```

## 설명
- Enhanced Input System을 사용하면 `PlayerInput`의 실체가 이 클래스가 된다. `UPlayerInput`을 상속받은 클래스의 `InputKey`가 전부 호출되는 구조이므로, 먼저 `Super::InputKey()`([[UPlayerInput/UPlayerInput.InputKey|UPlayerInput::InputKey]])로 기본 키 상태를 기록한 뒤 Enhanced Input 전용 처리를 얹는다.
- `KeysPressedThisTick`: 한 tick 안에서 눌렀다 뗀 입력(sub-tick tap)을 놓치지 않기 위한 기록이다. 입력은 메시지 단위로 들어오지만 처리는 tick 단위([[UEnhancedInputWorldSubsystem/UEnhancedInputWorldSubsystem.TickPlayerInput|TickPlayerInput]])이므로, 이 간극에서 짧은 탭이 사라질 수 있기 때문.
- `IE_Released` 분기는 flush된 키의 `bShouldBeIgnored`를 **release가 도착하는 즉시** 해제한다. tick 한 번을 기다렸다 해제하면 그 사이에 빠르게 다시 누른 입력이 ignored 상태로 조용히 먹혀서, "토글하려면 두 번 눌러야 한다"는 증상이 생긴다. UI를 여는 그 입력이 mapping rebuild로 flush를 유발할 때 발생하는 케이스.
