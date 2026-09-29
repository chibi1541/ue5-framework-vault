---
related:
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysis|FReachabilityAnalysisState::PerformReachabilityAnalysis]]"
  - "[[CollectGarbageImpl]]"
tags:
  - GarbageCollection_cpp
---

```cpp
AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageIncremental(EObjectFlags KeepFlags)
{
	CollectGarbageImpl<false>(KeepFlags);
}
```

## 설명
- [[CollectGarbageImpl]]을 `bPerformFullPurge = false`로 instantiate해 호출하는 wrapper.
- `FORCENOINLINE`으로 선언해 profiler에서 incremental GC와 full GC 호출이 구분되어 보이도록 한다.
