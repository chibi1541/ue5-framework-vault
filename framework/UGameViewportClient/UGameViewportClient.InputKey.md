---
related:
  - "[[UCommonGameViewportClient/UCommonGameViewportClient.InputKey|UCommonGameViewportClient::InputKey]]"
  - "[[APlayerController/APlayerController.InputKey|APlayerController::InputKey]]"
tags:
  - GameViewportClient_cpp
---

```cpp
bool UGameViewportClient::InputKey(const FInputKeyEventArgs& InEventArgs)
{

	// ...

	if (!bResult)
	{
		ULocalPlayer* const TargetPlayer = GEngine->GetLocalPlayerFromInputDevice(this, EventArgs.InputDevice);
		if (TargetPlayer && TargetPlayer->PlayerController)
		{
			bResult = TargetPlayer->PlayerController->InputKey(EventArgs);
		}

		// A gameviewport is always considered to have responded to a mouse buttons to avoid throttling
		if (!bResult && EventArgs.Key.IsMouseButton())
		{
			bResult = true;
		}
	}
	
	return bResult;
}

```

## 설명
- 입력이 **뷰포트 레벨에서 플레이어 단위로 갈라지는 지점**이다.
- `GEngine->GetLocalPlayerFromInputDevice()`로 해당 입력 장치를 소유한 `ULocalPlayer`를 찾는다. 로컬 멀티플레이(split screen)에서 어떤 패드가 어떤 플레이어의 입력인지 구분하는 역할.
- 찾은 플레이어의 `PlayerController`로 [[APlayerController/APlayerController.InputKey|InputKey()]]를 넘긴다. 여기서부터 게임플레이 프레임워크 영역이다.
- 마지막의 마우스 버튼 예외 처리: 아무도 처리하지 않았어도 마우스 버튼이면 `true`로 만든다. 뷰포트가 "응답 없음"으로 판정되어 엔진이 프레임을 throttling하는 것을 막기 위한 장치다.
