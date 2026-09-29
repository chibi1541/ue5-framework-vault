---
related:
  - "[[CollectGarbageInternal]]"
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysisAndConditionallyPurgeGarbage|FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void FReachabilityAnalysisState::CollectGarbage(EObjectFlags KeepFlags, bool bFullPurge)
{
	using namespace UE::GC::Private;

	if (GIsIncrementalReachabilityPending)
	{
		// Something triggered a new GC run but we're in the middle of incremental reachability analysis.
		// Finish the current GC pass (including purging all unreachable objects) and then kick off another GC run as requested
		bPerformFullPurge = true;
		PerformReachabilityAnalysisAndConditionallyPurgeGarbage(/*bReachabilityUsingTimeLimit =*/ false);

		checkf(!GIsIncrementalReachabilityPending, TEXT("Flushing incremental reachability analysis did not complete properly"));

		// Need to acquire GC lock again as it was released in PerformReachabilityAnalysisAndConditionallyPurgeGarbage() -> UE::GC::PostCollectGarbageImpl()
		AcquireGCLock();
	}

	ObjectKeepFlags = KeepFlags;
	bPerformFullPurge = bFullPurge;

	const bool bReachabilityUsingTimeLimit = !bFullPurge && GAllowIncrementalReachability;
	PerformReachabilityAnalysisAndConditionallyPurgeGarbage(bReachabilityUsingTimeLimit);
}
```

## 설명
- incremental reachability analysis가 진행 중일 때 새 GC 요청이 들어오면, 먼저 진행 중인 pass를 full purge로 끝까지 마친 뒤(time limit 없이) GC lock을 다시 잡고 새 GC를 시작한다.
  - lock을 다시 잡는 이유: `PostCollectGarbageImpl()` 안에서 lock이 해제되기 때문.
- `ObjectKeepFlags`, `bPerformFullPurge`를 state에 저장한다.
- full purge가 아니고 `GAllowIncrementalReachability`가 켜져 있으면 time limit을 사용하는 incremental 방식으로 [[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysisAndConditionallyPurgeGarbage|PerformReachabilityAnalysisAndConditionallyPurgeGarbage()]]를 호출한다.
