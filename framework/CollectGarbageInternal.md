---
related:
  - "[[TryCollectGarbage]]"
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.CollectGarbage|FReachabilityAnalysisState::CollectGarbage]]"
tags:
  - GarbageCollection_cpp
---

```cpp
AUTORTFM_DISABLE FORCEINLINE void CollectGarbageInternal(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	const double StartTime = FPlatformTime::Seconds();

	GReachabilityState.CollectGarbage(KeepFlags, bPerformFullPurge);

	GTimingInfo.LastGCDuration = FPlatformTime::Seconds() - StartTime;

	CSV_CUSTOM_STAT(GC, Count, 1, ECsvCustomStatOp::Accumulate);

}
```

## 설명
- [[TryCollectGarbage]]에서 GC lock을 획득한 뒤 호출된다.
- 전역 `GReachabilityState`의 [[FReachabilityAnalysisState/FReachabilityAnalysisState.CollectGarbage|FReachabilityAnalysisState::CollectGarbage()]]에 실제 작업을 위임한다.
- 소요 시간을 `GTimingInfo.LastGCDuration`에 기록하고 GC 횟수를 CSV stat으로 누적한다.
