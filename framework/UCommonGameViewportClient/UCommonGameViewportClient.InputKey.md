---
related:
  - "[[FSceneViewport/FSceneViewport.OnKeyDown|FSceneViewport::OnKeyDown]]"
  - "[[UGameViewportClient/UGameViewportClient.InputKey|UGameViewportClient::InputKey]]"
tags:
  - CommonGameViewportClient_cpp
---

```cpp
bool UCommonGameViewportClient::InputKey(const FInputKeyEventArgs& InEventArgs)
{
	FInputKeyEventArgs EventArgs = InEventArgs;

	if (IsKeyPriorityAboveUI(EventArgs))
	{
		return true;
	}

	// Check override before UI
	if (OnOverrideInputKey().IsBound())
	{
		if (OnOverrideInputKey().Execute(EventArgs))
		{
			return true;
		}
	}

	// The input is fair game for handling - the UI gets first dibs
#if ALLOW_CONSOLE
	if (ViewportConsole && !ViewportConsole->ConsoleState.IsEqual(NAME_Typing) && !ViewportConsole->ConsoleState.IsEqual(NAME_Open))
#endif
	{		
		FReply Result = FReply::Unhandled();
		if (!OnRerouteInput().ExecuteIfBound(EventArgs.InputDevice, EventArgs.Key, EventArgs.Event, Result))
		{
			HandleRerouteInput(EventArgs.InputDevice, EventArgs.Key, EventArgs.Event, Result);
		}

		if (Result.IsEventHandled())
		{
			return true;
		}
	}
	
	// 여기가 UGameViewportClient::InputKey()
	return Super::InputKey(EventArgs);
}
```

## 설명
- CommonUI 플러그인이 제공하는 viewport client. UI(특히 게임패드 focus 기반 UI)에 입력 우선권을 주기 위해 [[UGameViewportClient/UGameViewportClient.InputKey|UGameViewportClient::InputKey]] 앞에 끼어든다.
- 처리 순서는 세 단계다.
  1. `IsKeyPriorityAboveUI()` — UI보다도 우선하는 키(예: 콘솔 토글 같은 특수 키)면 즉시 소비.
  2. `OnOverrideInputKey()` — 바인딩된 override 델리게이트가 있으면 먼저 기회를 준다.
  3. `HandleRerouteInput()` / `OnRerouteInput()` — 입력을 UI 쪽으로 rerouting해서 UI가 먼저 가져가게 한다.
- 콘솔이 열려 있거나 타이핑 중(`NAME_Open` / `NAME_Typing`)이면 rerouting 단계를 건너뛴다. 콘솔 입력이 UI에 먹히면 안 되기 때문.
- 위 단계에서 아무도 소비하지 않으면 `Super::InputKey()`, 즉 [[UGameViewportClient/UGameViewportClient.InputKey|UGameViewportClient::InputKey]]로 내려가 게임 로직이 처리한다.
