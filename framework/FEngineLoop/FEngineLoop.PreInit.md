---
related:
  - "[[EnginePreInit]]"
  - "[[FEngineLoop/FEngineLoop.Init|FEngineLoop::Init]]"
tags:
  - LaunchEngineLoop_cpp
---

```cpp
/** pre-initialize the main loop - parse command line, sets up GIsEditor, etc */
int32 PreInit(const TCHAR* CmdLine)
{
    //...
}
```

## 설명
- 메인 루프의 pre-initialize 단계.
- command line을 parse하고 `GIsEditor` 등을 설정한다.
