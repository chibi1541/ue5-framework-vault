---
related:
  - "[[FSlateApplication/FSlateApplication.ProcessKeyDownEvent|FSlateApplication::ProcessKeyDownEvent]]"
  - "[[FSceneViewport/FSceneViewport.OnKeyDown|FSceneViewport::OnKeyDown]]"
tags:
  - SViewport_cpp
---

```cpp
FReply SViewport::OnKeyDown( const FGeometry& MyGeometry, const FKeyEvent& KeyEvent )
{
	return ViewportInterface.IsValid() ? ViewportInterface.Pin()->OnKeyDown(MyGeometry, KeyEvent) : FReply::Unhandled();
}
```

## 설명
- 게임 화면을 담는 Slate 위젯. [[FSlateApplication/FSlateApplication.ProcessKeyDownEvent|ProcessKeyDownEvent]]의 bubbling 과정에서 UI 위젯들이 소비하지 않은 입력이 여기로 들어온다.
- 자기가 직접 처리하지 않고 `ViewportInterface`(weak ptr)로 그대로 위임한다. 실제 구현체가 [[FSceneViewport/FSceneViewport.OnKeyDown|FSceneViewport]]다.
- `ViewportInterface`가 유효하지 않으면 `FReply::Unhandled()`를 반환해 상위 위젯으로 계속 bubbling되게 둔다.
- 이 한 단계 덕분에 Slate 위젯(`SViewport`)과 뷰포트 구현(`FSceneViewport`)이 분리된다.
