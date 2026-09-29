---
related:
  - "[[CollectGarbageImpl]]"
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysisAndConditionallyPurgeGarbage|FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage]]"
  - "[[FRealtimeGC/FRealtimeGC.StartReachabilityAnalysis|FRealtimeGC::StartReachabilityAnalysis]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void PerformReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	if (!GReachabilityState.IsSuspended())
	{
		StartReachabilityAnalysis(KeepFlags, Options);
		// We start verse GC here so that the objects are unmarked prior to verse marking them
		StartVerseGC();
	}
	
	{
		const double StartTime = FPlatformTime::Seconds();
		while (true)
		{
			PerformReachabilityAnalysisPass(Options);

			if (GReachabilityState.IsSuspended())
			{
				// We may have suspended either via incremental timeout, or because verse GC is still marking.
				// If we are not incremental at all, keep going while verse GC adds to GReachableObjects.
				// If we are incremental without a time limit, the goal is still to reach all objects, so never stop early.
				if (EnumHasAnyFlags(Options, EGCOptions::IncrementalReachability) && GReachabilityState.IsTimeLimitExceeded())
				{
					break;
				}
			}
			else if (Private::GReachableObjects.IsEmpty()
				&& Private::GReachableClusters.IsEmpty()
#if WITH_VERSE_VM || defined(__INTELLISENSE__)
				&& Private::GReachableNativeStructs.IsEmpty()
#endif					
				)
			{
				// We terminate verse GC here now that both sides have nothing left to mark.
				// This check must happen only when !IsSuspended, so verse GC can no longer add to GReachableObjects.
				StopVerseGC();
				break;
			}
		}

		const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
		if (!bIsGarbageTracking)
		{
			GGCStats.ReferenceCollectionTime += ElapsedTime;
		}
		UE_LOGF(LogGarbage, Verbose, "%f ms for Reachability Analysis", ElapsedTime * 1000);
	}
}
```

## 설명
- suspend된 상태(이전 incremental pass의 이어하기)가 아니면 [[FRealtimeGC/FRealtimeGC.StartReachabilityAnalysis|StartReachabilityAnalysis()]]로 초기 marking을 하고 Verse GC를 시작한다.
- 이후 `PerformReachabilityAnalysisPass()`를 반복해 참조를 따라가며 reachable object를 marking한다.
  - suspend 상태: incremental 옵션이고 time limit을 초과했으면 루프를 빠져나와 다음 프레임에 이어서 진행한다.
  - suspend가 아니고 `GReachableObjects`, `GReachableClusters`가 모두 비었으면 더 marking할 대상이 없으므로 Verse GC를 멈추고 종료한다.
- garbage tracking rerun(`bIsGarbageTracking`)이 아닐 때만 소요 시간을 `GGCStats.ReferenceCollectionTime`에 누적한다.
