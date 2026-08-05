---
summarize: true
---

#### Foundation - Input - class GenericApplication(GenericApplication.h)

```cpp
class GenericApplication
{
	TSharedRef< class FGenericApplicationMessageHandler > MessageHandler;
}
```


#### Foundation - Input - Begin FWindowsApplication::ProcessDeferredMessages(WindowsApplication.cpp)

```cpp
int32 FWindowsApplication::ProcessDeferredMessage( const FDeferredWindowsMessage& DeferredMessage )
{
	// ....
	// 윈도우 메시지 처리
	// 언리얼 엔진도 기본적으로 Windows App이기 때문에 WinAPI와 같은 작동방식
	switch(msg)
	{
		// ...
		case WM_KEYDOWN:
		{
			// ...
			// ActualKey에 KeyCode가 들어옴
			const bool Result = MessageHandler->OnKeyDown(ActualKey, CharCode, bIsRepeat)
		}
	}
	
}
```

#### Foundation - Input - FSlateApplication::OnKeyDown(SlateApplication.cpp)

언리얼의 입력처리는 플랫폼에서 입력을 받아서 Slate에서 UI 먼저 입력을 처리할 수
있는지를 판단 후 없다면 게임 로직으로 전달하는 방식
```cpp
bool FSlateApplication::OnKeyDown(const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat)
{
	FKey const key = FInputKeyManager::Get().GetKeyFromCodes(KeyCode, CharacterCode);
	FKeyEvent KeyEvent(Key, PlatformApplication->GetModifierKeys(), GetUserIndexForkeyboard(), IsRepeat, CharacterCode, KeyCode);
	
	return ProcessKeyDownEvent(KeyEvent);
}
```

#### Foundation - Input - FSlateApplication::PressKeyDownEvent(SlateApplication.cpp)

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

#### Foundation - Input - SViewport::OnKeyDown(SViewport.cpp)

```cpp
FReply SViewport::OnKeyDown( const FGeometry& MyGeometry, const FKeyEvent& KeyEvent )
{
	return ViewportInterface.IsValid() ? ViewportInterface.Pin()->OnKeyDown(MyGeometry, KeyEvent) : FReply::Unhandled();
}
```

#### Foundation - Input - FSceneViewport::OnKeyDown(SceneViewport.cpp)

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

#### Foundation - Input - UCommonGameViewportClient::InputKey(CommonGameViewportClient.cpp)

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

#### Foundation - Input - UGameViewportClient::InputKey(GameViewportClient.cpp)
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

#### Foundation - Input - APlayerController::InputKey(PlayerController.cpp)

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

#### Foundation - Input - UEnhancedPlayerInput::InputKey(EnhancedPlayerInput.cpp)
UPlayerInput을 상속받은 클래스(EnhancedInputSystem을 사용하게 되면 UEnhancedPlayerInput)의 InputKey가 전부 호출됨

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

#### Foundation - Input - UPlayerInput::InputKey(PlayerInput.cpp)

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

#### Foundation - Input - Begin UEnhancedInputWorldSubsystem::TickPlayerInput(EnhancedInputWorldSubsystem.cpp)

이번 프레임에 입력 받은 input 이벤트를 Tick에 한번에 처리
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

#### Foundation - Input -  UPlayerInput::ProcessInputStack(PlayerInput.cpp)
```cpp
void UPlayerInput::ProcessInputStack(const TArray<UInputComponent*>& InputComponentStack, const float DeltaTime, const bool bGamePaused)
{
	static TArray<TPair<FKey, FKeyState*>> KeysWithEvents;

// Evaluate the current state of the key map. This will take the event accumulators and put them into the actual values of the key state map.
	EvaluateKeyMapState(DeltaTime, bGamePaused, OUT KeysWithEvents);
	
	// Determine potential delegates
	EvaluateInputDelegates(InputComponentStack, DeltaTime, bGamePaused, KeysWithEvents);
}

```

#### Foundation - Input -  UPlayerInput::EvaluateKeyMapState(PlayerInput.cpp)

```cpp
void UPlayerInput::EvaluateKeyMapState(const float DeltaTime, const bool bGamePaused, OUT TArray<TPair<FKey, FKeyState*>>& KeysWithEvents)
{
	for (TMap<FKey,FKeyState>::TIterator It(KeyStateMap); It; ++It)
	{
		bool bKeyHasEvents = false;
		FKeyState* const KeyState = &It.Value();
		const FKey& Key = It.Key();

		for (uint8 EventIndex = 0; EventIndex < IE_MAX; ++EventIndex)
		{
			KeyState->EventCounts[EventIndex].Reset();
			Exchange(KeyState->EventCounts[EventIndex], KeyState->EventAccumulator[EventIndex]);

			if (!bKeyHasEvents && KeyState->EventCounts[EventIndex].Num() > 0)
			{
				KeysWithEvents.Emplace(Key, KeyState);
				bKeyHasEvents = true;
			}
		}

		if ( (KeyState->SampleCountAccumulator > 0) || Key.ShouldUpdateAxisWithoutSamples() )
		{
			if (KeyState->PairSampledAxes)
			{
				// Paired keys sample only the axes that have changed, leaving unaltered axes in their previous state
				for (int32 Axis = 0; Axis < 3; ++Axis)
				{
					if (KeyState->PairSampledAxes & (1 << Axis))
					{
						KeyState->RawValue[Axis] = KeyState->RawValueAccumulator[Axis];
					}
				}
			}
			else
			{
				// Unpaired keys just take the whole accumulator
				KeyState->RawValue = KeyState->RawValueAccumulator;
			}

			// if we had no samples, we'll assume the state hasn't changed
			// except for some axes, where no samples means the mouse stopped moving
			if (KeyState->SampleCountAccumulator == 0)
			{
				KeyState->EventCounts[IE_Released].Add(++EventCount);
				if (!bKeyHasEvents)
				{
					KeysWithEvents.Emplace(Key, KeyState);
					bKeyHasEvents = true;
				}
			}
		}
	}
	
	

}
```

#### Foundation - Input -  UPlayerInput::EvaluateInputDelegates(PlayerInput.cpp)
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

#### Foundation - Input -  UPlayerInput::EvaluateInputComponentDelegates(PlayerInput.cpp)
```cpp
bool UPlayerInput::EvaluateInputComponentDelegates(UInputComponent* const IC, const TArray<TPair<FKey, FKeyState*>>& KeysWithEvents, const float DeltaTime, const bool bGamePaused)
{
	

}
```