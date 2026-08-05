---
related:
  - "[[FSlateApplication/FSlateApplication.OnKeyDown|FSlateApplication::OnKeyDown]]"
  - "[[SViewport/SViewport.OnKeyDown|SViewport::OnKeyDown]]"
tags:
  - SlateApplication_cpp
---

```cpp
bool FSlateApplication::ProcessKeyDownEvent(const FKeyEvent& InKeyEvent)
{
	// 위젯에서 이벤트 처리
	//...
	
	// 여기까지 Reply가 true 상태가 아니라는건 Widget쪽에서 처리되는 입력이 아니라는 의미
// Send out key down events.
		if ( !Reply.IsEventHandled() )
		{
			Reply = FEventRouter::RouteAlongFocusPath(this, FEventRouter::FBubblePolicy(EventPath), InKeyEvent, [] (const FArrangedWidget& SomeWidgetGettingEvent, const FKeyEvent& Event)
			{
				if (SomeWidgetGettingEvent.Widget->IsEnabled())
				{
					// 여기가 SViewport의 OnKeyDown이벤트가 들어옴
					const FReply TempReply = SomeWidgetGettingEvent.Widget->OnKeyDown(SomeWidgetGettingEvent.Geometry, Event);
#if WITH_SLATE_DEBUGGING
					FSlateDebugging::BroadcastInputEvent(ESlateDebuggingInputEvent::KeyDown, &Event, TempReply, SomeWidgetGettingEvent.Widget, Event.GetKey().GetFName());
#endif
					return TempReply;
				}
				else
				{
#if WITH_SLATE_DEBUGGING
					FSlateDebugging::BroadcastNoReplyInputEvent(ESlateDebuggingInputEvent::KeyDown, &Event, SomeWidgetGettingEvent.Widget);
#endif
				}

				return FReply::Unhandled();
			}, ESlateDebuggingInputEvent::KeyDown);
		}
	
}

```

## 설명
- Slate가 키 입력을 위젯 계층에 태워 보내는 함수. **UI가 먼저 입력을 가져갈 기회**를 주는 단계다.
- `Reply.IsEventHandled()`가 false라는 것은 앞선 처리에서 아무 위젯도 이 입력을 소비하지 않았다는 뜻이다. 이때 `FEventRouter::RouteAlongFocusPath`로 focus path를 따라 이벤트를 전달한다.
- `FBubblePolicy`이므로 **bubbling**(가장 안쪽 위젯 → 바깥쪽 부모 순)으로 전파되며, 어느 위젯이 `FReply::Handled()`를 반환하면 거기서 멈춘다. `IsEnabled()`가 false인 위젯은 건너뛴다.
- 게임 화면은 결국 `SViewport` 위젯이므로, UI 위젯들이 소비하지 않은 입력은 [[SViewport/SViewport.OnKeyDown|SViewport::OnKeyDown]]으로 들어온다. 여기가 Slate → 게임 로직으로 넘어가는 경계다.
