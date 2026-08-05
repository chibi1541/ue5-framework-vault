---
related:
  - "[[FWindowsApplication/FWindowsApplication.ProcessDeferredMessage|FWindowsApplication::ProcessDeferredMessage]]"
  - "[[FSlateApplication/FSlateApplication.ProcessKeyDownEvent|FSlateApplication::ProcessKeyDownEvent]]"
  - "[[GenericApplication/GenericApplication|GenericApplication]]"
tags:
  - SlateApplication_cpp
---

```cpp
bool FSlateApplication::OnKeyDown(const int32 KeyCode, const uint32 CharacterCode, const bool IsRepeat)
{
	FKey const key = FInputKeyManager::Get().GetKeyFromCodes(KeyCode, CharacterCode);
	FKeyEvent KeyEvent(Key, PlatformApplication->GetModifierKeys(), GetUserIndexForkeyboard(), IsRepeat, CharacterCode, KeyCode);
	
	return ProcessKeyDownEvent(KeyEvent);
}
```

## 설명
- 언리얼의 입력 처리는 **플랫폼에서 입력을 받아 → Slate가 UI에서 먼저 처리할 수 있는지 판단 → 처리하지 못하면 게임 로직으로 전달**하는 방식이다. 이 함수가 그 Slate 단계의 진입점이다.
- 플랫폼이 넘겨준 raw한 `KeyCode`/`CharacterCode`를 `FInputKeyManager::GetKeyFromCodes()`로 엔진 공용 표현인 `FKey`로 변환한다. 이 지점부터는 플랫폼 의존성이 사라진다.
- 변환된 `FKey`에 modifier 키 상태, 키보드 유저 인덱스, repeat 여부 등을 묶어 `FKeyEvent`를 만들고 [[FSlateApplication/FSlateApplication.ProcessKeyDownEvent|ProcessKeyDownEvent]]로 넘긴다.
