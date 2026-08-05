---
related:
  - "[[FWindowsApplication/FWindowsApplication.ProcessDeferredMessage|FWindowsApplication::ProcessDeferredMessage]]"
  - "[[FSlateApplication/FSlateApplication.OnKeyDown|FSlateApplication::OnKeyDown]]"
tags:
  - GenericApplication_h
---

```cpp
class GenericApplication
{
	TSharedRef< class FGenericApplicationMessageHandler > MessageHandler;
}
```

## 설명
- 플랫폼별 application(윈도우면 `FWindowsApplication`)의 공통 base. 플랫폼에서 올라온 입력/윈도우 메시지를 상위로 전달하는 통로 역할을 한다.
- 핵심 멤버는 `MessageHandler` 하나다. 플랫폼 레이어는 이 인터페이스만 알고 `MessageHandler->OnKeyDown(...)` 식으로 호출하며, 실제 구현체로는 [[FSlateApplication/FSlateApplication.OnKeyDown|FSlateApplication]]이 꽂힌다.
- 즉 이 클래스가 **플랫폼 레이어와 Slate 레이어를 분리하는 경계**다. 플랫폼은 Slate를 직접 알지 않고, Slate는 WinAPI를 직접 알지 않는다.
