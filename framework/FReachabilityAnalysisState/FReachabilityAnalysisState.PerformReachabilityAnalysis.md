---
related:
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysisAndConditionallyPurgeGarbage|FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage]]"
  - "[[CollectGarbageFull]]"
  - "[[CollectGarbageIncremental]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void FReachabilityanalysisState::PerformReachabilityAnalysis()
{
	if (bPerformFullPurge)
	{
		UE::GC::CollectGarbageFull(ObjectKeepFlags);
	}
	else if (NumRechabilityIterationsToSkip == 0 || // Delay reachability analysis by NumRechabilityIterationsToSkip (if desired)
		!bIsSuspended || // but only but only after the first iteration (which also does MarkObjectsAsUnreachable)
		IterationTimeLimit <= 0.0f) // and only when using time limit (we're not using the limit when we're flushing reachability analysis when starting a new one or on exit)
	{
		UE::GC::CollectGarbageIncremental(ObjectKeepFlags);
	}
	else
	{
		--NumRechabilityIterationsToSkip;
	}

	FinishIteration();
}
```

## 설명
- full purge면 [[CollectGarbageFull]], 아니면 [[CollectGarbageIncremental]]을 호출한다.
- incremental인 경우 `NumRechabilityIterationsToSkip`만큼 analysis를 지연시킬 수 있다. 단, 첫 iteration(`!bIsSuspended`, `MarkObjectsAsUnreachable` 포함)이거나 time limit을 쓰지 않는 경우(`IterationTimeLimit <= 0`)에는 지연 없이 바로 실행한다.
- 마지막에 `FinishIteration()`으로 이번 iteration을 마무리한다.
