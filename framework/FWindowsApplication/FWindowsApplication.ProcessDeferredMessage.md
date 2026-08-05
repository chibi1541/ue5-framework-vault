---
related:
  - "[[GenericApplication/GenericApplication|GenericApplication]]"
  - "[[FSlateApplication/FSlateApplication.OnKeyDown|FSlateApplication::OnKeyDown]]"
tags:
  - WindowsApplication_cpp
---

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

## 설명
- 입력 처리 흐름의 **시작점**. 언리얼 엔진도 결국 Windows 애플리케이션이므로, WinAPI의 메시지 루프와 동일한 방식으로 윈도우 메시지를 `switch`로 처리한다.
- 이름이 `Deferred`인 이유는 윈도우 프로시저(WndProc)에서 즉시 처리하지 않고 메시지를 큐에 쌓아 두었다가, 엔진이 원하는 타이밍에 꺼내서 처리하기 때문이다.
- `WM_KEYDOWN`에서 키 코드(`ActualKey`), 문자 코드(`CharCode`), 반복 여부(`bIsRepeat`)를 뽑아 [[GenericApplication/GenericApplication|GenericApplication]]의 `MessageHandler`로 넘긴다. 이 `MessageHandler`가 [[FSlateApplication/FSlateApplication.OnKeyDown|FSlateApplication::OnKeyDown]]이다.
