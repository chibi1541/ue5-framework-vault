---
related:
  - "[[UGameViewportClient/UGameViewportClient.InputKey|UGameViewportClient::InputKey]]"
  - "[[UEnhancedPlayerInput/UEnhancedPlayerInput.InputKey|UEnhancedPlayerInput::InputKey]]"
  - "[[UPlayerInput/UPlayerInput.InputKey|UPlayerInput::InputKey]]"
tags:
  - PlayerController_cpp
---

```cpp
bool APlayerController::InputKey(const FInputKeyEventArgs& Params)
{
	bool bResult = false;

	// Only process the given input if it came from an input device that is owned by our owning local player
	if (GetDefault<UInputSettings>()->bFilterInputByPlatformUser &&
		IPlatformInputDeviceMapper::Get().GetUserForInputDevice(Params.InputDevice) != GetPlatformUserId())
	{
		return false;
	}
	
	// Any analog values can simply be passed to the UPlayerInput
	if(Params.Key.IsAnalog())
	{
		
	}
	else
	{
		if (PlayerInput)
		{
			bResult = PlayerInput->InputKey(Params);
			if (bEnableClickEvents && (ClickEventKeys.Contains(Params.Key) || ClickEventKeys.Contains(EKeys::AnyKey)))
			{
				FVector2D MousePosition;
				UGameViewportClient* ViewportClient = CastChecked<ULocalPlayer>(Player)->ViewportClient;
				if (ViewportClient && ViewportClient->GetMousePosition(MousePosition))
				{
					UPrimitiveComponent* ClickedPrimitive = nullptr;
					if (bEnableMouseOverEvents)
					{
						ClickedPrimitive = CurrentClickablePrimitive.Get();
					}
					else
					{
						FHitResult HitResult;
						const bool bHit = GetHitResultAtScreenPosition(MousePosition, CurrentClickTraceChannel, true, HitResult);
						if (bHit)
						{
							ClickedPrimitive = HitResult.Component.Get();
						}
					}
					if(GetHUD())
					{
						if (GetHUD()->UpdateAndDispatchHitBoxClickEvents(MousePosition, Params.Event))
						{
							ClickedPrimitive = nullptr;
						}
					}

					if (ClickedPrimitive)
					{
						switch(Params.Event)
						{
						case IE_Pressed:
						case IE_DoubleClick:
							ClickedPrimitive->DispatchOnClicked(Params.Key);
							break;

						case IE_Released:
							ClickedPrimitive->DispatchOnReleased(Params.Key);
							break;

						case IE_Axis:
						case IE_Repeat:
							break;
						}
					}

					bResult = true;
				}
			}
		}
	}
	
	return bResult;
}
```

## 설명
- 앞단의 필터: `bFilterInputByPlatformUser`가 켜져 있으면, 이 컨트롤러를 소유한 platform user의 장치에서 온 입력이 아닐 경우 그냥 버린다. 로컬 멀티플레이에서 남의 패드 입력이 섞이는 것을 막는다.
- 핵심은 `PlayerInput->InputKey(Params)` 한 줄이다. 실제 키 상태 기록은 `UPlayerInput`(Enhanced Input 사용 시 [[UEnhancedPlayerInput/UEnhancedPlayerInput.InputKey|UEnhancedPlayerInput]])이 담당한다.
- 나머지 긴 블록은 전부 **클릭 이벤트(`OnClicked` / `OnReleased`) 디스패치** 처리다. `bEnableClickEvents`가 켜져 있고 해당 키가 `ClickEventKeys`에 등록돼 있을 때만 동작한다.
  - `bEnableMouseOverEvents`가 켜져 있으면 이미 hover 판정으로 구해 둔 `CurrentClickablePrimitive`를 재사용하고, 아니면 그 자리에서 `GetHitResultAtScreenPosition()`으로 화면 좌표를 trace해 컴포넌트를 찾는다. 매 클릭마다 trace를 새로 쏘는 비용을 아끼는 구조.
  - HUD의 hit box가 먼저 클릭을 가져가면 `ClickedPrimitive`를 `nullptr`로 지워, 3D 오브젝트와 HUD 클릭이 중복 처리되지 않게 한다.
- analog 키는 별도 분기로 빠져 `UPlayerInput`에 그대로 전달된다.
