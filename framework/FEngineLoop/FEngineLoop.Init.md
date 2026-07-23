---
related:
  - "[[EditorInit]]"
  - "[[FEngineLoop/FEngineLoop.PreInit|FEngineLoop::PreInit]]"
tags:
  - LaunchEngineLoop_cpp
---

```cpp
int32 FEngineLoop::Init()
{

#if WITH_EDITOR
		GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
#else

		GEngine->Init(this);
}
```

## 설명
- Editor 빌드에서는 `UUnrealEdEngine`을 생성하여 `GEngine`, `GEditor`, `GUnrealEd` 전역에 대입한다.
- 그 외 빌드에서는 `GEngine->Init()`을 호출한다.
