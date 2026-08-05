---
related:
  - "[[SViewport/SViewport.OnKeyDown|SViewport::OnKeyDown]]"
  - "[[UCommonGameViewportClient/UCommonGameViewportClient.InputKey|UCommonGameViewportClient::InputKey]]"
tags:
  - SceneViewport_cpp
---

```cpp
FReply FSceneViewport::OnKeyDown( const FGeometry& InGeometry, const FKeyEvent& InKeyEvent )
{
	// Start a new reply state
	CurrentReplyState = FReply::Handled(); 

	FKey Key = InKeyEvent.GetKey();
	if (Key.IsValid())
	{
		
		KeyStateMap.Add(Key, true);

		//@todo Slate Viewports: FWindowsViewport checks for Alt+Enter or F11 and toggles fullscreen.  Unknown if fullscreen via this method will be needed for slate viewports. 
PRAGMA_DISABLE_DEPRECATION_WARNINGS
		FViewportClient* ClientPtr = ViewportClient;
PRAGMA_ENABLE_DEPRECATION_WARNINGS
		if (ClientPtr && GetSizeXY() != FIntPoint::ZeroValue)
		{
			// Switch to the viewport clients world before processing input
			FScopedConditionalWorldSwitcher WorldSwitcher(ClientPtr);

			if (!ClientPtr->InputKey(FInputKeyEventArgs(
				this, InKeyEvent.GetInputDeviceId(), Key, InKeyEvent.IsRepeat() ? IE_Repeat : IE_Pressed, 1.0f, false, InKeyEvent.GetEventTimestamp())))
			{
				CurrentReplyState = FReply::Unhandled();
			}
		}
	}
	else
	{
		CurrentReplyState = FReply::Unhandled();
	}
	return CurrentReplyState;
}

```

## 설명
- [[SViewport/SViewport.OnKeyDown|SViewport::OnKeyDown]]이 위임한 실제 구현. 여기서 Slate의 `FKeyEvent`가 게임 입력 표현인 `FInputKeyEventArgs`로 바뀐다.
- 일단 `FReply::Handled()`로 시작해 두고, 뷰포트 클라이언트가 처리하지 못했을 때만 `Unhandled()`로 되돌리는 낙관적(optimistic) 패턴이다.
- `KeyStateMap.Add(Key, true)`로 뷰포트 자체의 키 눌림 상태도 따로 기록한다.
- `FScopedConditionalWorldSwitcher`는 입력을 처리하기 전에 해당 viewport client의 world로 전환한다. 에디터처럼 여러 world가 공존하는 환경에서 `GetWorld()`가 엉뚱한 world를 가리키지 않게 하는 장치다.
- `IsRepeat()`에 따라 `IE_Repeat` / `IE_Pressed`로 이벤트 종류를 결정해 [[UCommonGameViewportClient/UCommonGameViewportClient.InputKey|ViewportClient->InputKey()]]로 넘긴다.
