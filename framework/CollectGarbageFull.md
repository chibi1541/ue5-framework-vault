---
related:
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysis|FReachabilityAnalysisState::PerformReachabilityAnalysis]]"
  - "[[CollectGarbageImpl]]"
tags:
  - GarbageCollection_cpp
---

```cpp
AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageFull(EObjectFlags KeepFlags)
{
	CollectGarbageImpl<true>(KeepFlags);
}
```

## 설명
- [[CollectGarbageImpl]]을 `bPerformFullPurge = true`로 instantiate해 호출하는 wrapper.
- [[CollectGarbageIncremental]]과 쌍을 이루며, full purge 요청 시 사용된다.
