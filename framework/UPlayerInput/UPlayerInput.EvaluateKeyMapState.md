---
related:
  - "[[UPlayerInput/UPlayerInput.ProcessInputStack|UPlayerInput::ProcessInputStack]]"
  - "[[UPlayerInput/UPlayerInput.InputKey|UPlayerInput::InputKey]]"
  - "[[UPlayerInput/UPlayerInput.EvaluateInputDelegates|UPlayerInput::EvaluateInputDelegates]]"
tags:
  - PlayerInput_cpp
---

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

## 설명
- [[UPlayerInput/UPlayerInput.InputKey|InputKey]]가 프레임 내내 쌓아 둔 accumulator를 이번 프레임의 **확정값**으로 옮기는 단계.
- `Exchange(EventCounts[i], EventAccumulator[i])`가 핵심이다. 복사가 아니라 **swap**이므로 확정값으로 옮기는 동시에 accumulator가 (직전에 `Reset()`된) 빈 배열로 비워진다. 한 번의 연산으로 "확정 + 초기화"가 끝나고 배열 재할당도 없다.
- 이벤트가 하나라도 있는 키는 `KeysWithEvents`에 담아 out으로 넘긴다. 다음 단계인 [[UPlayerInput/UPlayerInput.EvaluateInputDelegates|EvaluateInputDelegates]]가 전체 맵 대신 이것만 보면 되게 하기 위해서다.
- axis 값 처리에서 `PairSampledAxes`가 있으면 **변경된 축만** 갱신하고 나머지 축은 이전 값을 유지한다. paired key(예: 좌우 스틱이 하나의 2D 값을 이루는 경우)에서 한 축만 움직였는데 다른 축이 0으로 리셋되는 것을 막는다. paired가 아니면 accumulator 전체를 그대로 가져온다.
- `SampleCountAccumulator == 0`인데도 갱신 대상인 축(`ShouldUpdateAxisWithoutSamples()`)은 "샘플이 안 들어왔다 = 마우스가 멈췄다"로 해석해 `IE_Released`를 넣어준다. 마우스 이동은 멈추면 아예 이벤트가 오지 않기 때문에, 그 침묵 자체를 신호로 쓰는 것.
