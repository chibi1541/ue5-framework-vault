---
related:
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.CollectGarbage|FReachabilityAnalysisState::CollectGarbage]]"
  - "[[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysis|FReachabilityAnalysisState::PerformReachabilityAnalysis]]"
  - "[[FRealtimeGC/FRealtimeGC.PerformReachabilityAnalysis|FRealtimeGC::PerformReachabilityAnalysis]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage(bool bReachabilityUsingTimeLimit)
{
	// 증분적으로 GC를 하지 않고 모든 오브젝트에 대해 GC를 적용할 때
	if (!GIsIncrementalReachabilityPending)
	{
		GGCStats = UE::GC::Private::FStats();
		GGCStats.bInProgress = true;
		GGCStats.bStartedAsFullPurge = bPerformFullPurge;
		GGCStats.NumObjects = GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex();
		// FUObjectCluster : UObject들을 묶어 놓은 단위
		GGCStats.NumClusters = GUObjectClusters.GetNumAllocatedClusters();
		GGCStats.ReachabilityTimeLimit = GetReachabilityAnalysisTimeLimit();
	}
	
	if (bPerformFullPurge)
	{
		UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
	}
	else
	{
		UE::GC::PreCollectGarbageImpl<false>(ObjectKeepFlags);
	}
	
#if WITH_VERSE_VM || defined(__INTELLISENSE__)
	// Wait to relinquish access until after PreCollectGarbageImpl calls FlushAsyncLoading.
	Verse::FRunningContext RunningContext = Verse::FRunningContextPromise{};
	Verse::FIOContext Context = RelinquishVerseHeapAccess(RunningContext);
#endif
	
	if (bForceNonIncrementalReachability)
	{
		IncrementalMarkPhaseTotalTime = 0.0;
		ReferenceProcessingTotalTime = 0.0;
		PerformReachabilityAnalysis();
	}
	else
	{
		if (!GIsIncrementalReachabilityPending)
		{
			ReferenceProcessingTotalTime = 0.0;
			IncrementalMarkPhaseTotalTime = 0.0;
		}
		
		PerformReachabilityAnalysis();
	}
	
	
	if (!GIsIncrementalReachabilityPending && Stats.bFoundGarbageRef && GGarbageReferenceTrackingEnabled > 0)
	{
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(GarbageCollectionDebug);
		SCOPED_NAMED_EVENT(FRealtimeGC_PerformReachabilityAnalysisRerun, FColor::Orange);
		DECLARE_SCOPE_CYCLE_COUNTER(TEXT("FRealtimeGC::PerformReachabilityAnalysisRerun"), STAT_FArchiveRealtimeGC_PerformReachabilityAnalysisRerun, STATGROUP_GC);		
		const double StartTime = FPlatformTime::Seconds();
		{
			TGuardValue<bool> GuardReachabilityUsingTimeLimit(bReachabilityUsingTimeLimit, false);
			FRealtimeGC GC;
			GC.Stats = Stats; // This is to pass Stats.bFoundGarbageRef to CG
			GC.PerformReachabilityAnalysis(ObjectKeepFlags, GetReferenceCollectorOptions(bPerformFullPurge));
		}
		const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
		GGCStats.GarbageTrackingTime = ElapsedTime;
		GGCStats.TotalTime += ElapsedTime;
		UE_LOGF(LogGarbage, Log, "%.2f ms for GC rerun to track garbage references (gc.GarbageReferenceTrackingEnabled=%d)", ElapsedTime * 1000, GGarbageReferenceTrackingEnabled);
	}
	
	if (bPerformFullPurge)
	{
		UE::GC::PostCollectGarbageImpl<true>(ObjectKeepFlags);
	}
	else
	{
		UE::GC::PostCollectGarbageImpl<false>(ObjectKeepFlags);
	}
	
#if WITH_VERSE_VM || defined(__INTELLISENSE__)
	AcquireVerseHeapAccess(Context);
#endif
}
```

## 설명
- GC 한 pass의 전체 흐름: `PreCollectGarbageImpl` → reachability analysis → (필요 시 garbage tracking rerun) → `PostCollectGarbageImpl`.
- 새 GC pass가 시작될 때(`!GIsIncrementalReachabilityPending`)만 `GGCStats`를 초기화하고 object 수, cluster 수(`FUObjectCluster`: UObject들을 묶어 놓은 단위), time limit을 기록한다.
- `PreCollectGarbageImpl` / `PostCollectGarbageImpl`은 `bPerformFullPurge`에 따라 template 인자를 달리해 호출된다.
- 실제 reachability analysis는 [[FReachabilityAnalysisState/FReachabilityAnalysisState.PerformReachabilityAnalysis|PerformReachabilityAnalysis()]]가 담당한다. incremental 진행 중이 아닐 때만 누적 시간 값을 0으로 리셋한다.
- analysis가 끝났는데 garbage 참조가 발견되었고(`Stats.bFoundGarbageRef`) `gc.GarbageReferenceTrackingEnabled`가 켜져 있으면, time limit 없이 [[FRealtimeGC/FRealtimeGC.PerformReachabilityAnalysis|FRealtimeGC::PerformReachabilityAnalysis()]]를 한 번 더 실행해 garbage 참조를 추적한다.
