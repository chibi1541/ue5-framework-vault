---
summarize: true
---

#### Foundation - GarbageCollection - BEGIN(GarbageCollection.cpp)

``` cpp
bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	if (AutoRTFM::IsTransactional())
	{
		// Memory cannot be freed within a transaction as this would prevent us from rolling back to the initial state.
		UE_LOGF(LogGarbage, Log, "TryCollectGarbage: skipping garbage collection because an AutoRTFM transaction is active.");
		return false;
	}

	// No other thread may be performing UObject operations while we're running so try to acquire GC lock
	if (UE::GC::GIsIncrementalReachabilityPending)
	{
		// Since we're already in the middle of a previous GC acquire GC lock even if it means we have to block main thread
		AcquireGCLock();
	}	
	else if (!FGCCSyncObject::Get().TryGCLock())
	{
		if (GNumRetriesBeforeForcingGC > 0 && GNumAttemptsSinceLastGC > GNumRetriesBeforeForcingGC)
		{
			// Force acquire GC lock and block main thread		
			UE_LOGF(LogGarbage, Warning, "TryCollectGarbage: forcing GC after %d skipped attempts.", GNumAttemptsSinceLastGC);
			GNumAttemptsSinceLastGC = 0;
			AcquireGCLock();
		}
		else
		{
			++GNumAttemptsSinceLastGC;
			return false;
		}
	}

	// Perform actual garbage collection
	UE::GC::CollectGarbageInternal(KeepFlags, bPerformFullPurge);

	// GC lock was released after reachability analysis inside CollectGarbageInternal

	return true;

}
```

#### Foundation - GarbageCollection - CollectGarbageInternal(GarbageCollection.cpp)

```cpp
AUTORTFM_DISABLE FORCEINLINE void CollectGarbageInternal(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	const double StartTime = FPlatformTime::Seconds();

	GReachabilityState.CollectGarbage(KeepFlags, bPerformFullPurge);

	GTimingInfo.LastGCDuration = FPlatformTime::Seconds() - StartTime;

	CSV_CUSTOM_STAT(GC, Count, 1, ECsvCustomStatOp::Accumulate);

}

```

#### Foundation - GarbageCollection - FReachabilityAnalysisState::CollectGarbage(GarbageCollection.cpp)

``` cpp
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

#### Foundation - GarbageCollection - FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage(GarbageCollection.cpp)

``` cpp
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

#### Foundation - GarbageCollection - FReachabilityAnalysisState::PerformReachabilityAnalysis(GarbageCollection.cpp)

``` cpp
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

#### Foundation - GarbageCollection - CollectGarbageIncremental(GarbageCollection.cpp)

``` cpp
AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageIncremental(EObjectFlags KeepFlags)
{
	CollectGarbageImpl<false>(KeepFlags);
}
```

#### Foundation - GarbageCollection - CollectGarbageFull(GarbageCollection.cpp)

``` cpp
AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageFull(EObjectFlags KeepFlags)
{
	CollectGarbageImpl<true>(KeepFlags);
}
```

#### Foundation - GarbageCollection - CollectGarbageImpl(GarbageCollection.cpp)

``` cpp
template<bool bPerformFullPurge>
AUTORTFM_DISABLE void CollectGarbageImpl(EObjectFlags KeepFlags)
{
	{
		// Reachability analysis.
		{
			const EGCOptions Options = GetReferenceCollectorOptions(bPerformFullPurge);

			// Perform reachability analysis.
			FRealtimeGC GC;
			GC.PerformReachabilityAnalysis(KeepFlags, Options);
		}
	}
}
```

#### Foundation - GarbageCollection - FRealtimeGC::PerformReachabilityAnalysis(GarbageCollection.cpp)

``` cpp
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

#### Foundation - GarbageCollection - FRealtimeGC::StartReachabilityAnalysis(GarbageCollection.cpp)

``` cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	{
		const double StartTime = FPlatformTime::Seconds();
		MarkObjectsAsUnreachable(KeepFlags);
		const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
		if (!Stats.bFoundGarbageRef)
		{
			GGCStats.MarkObjectsAsUnreachableTime = ElapsedTime;
		}
		UE_LOGF(LogGarbage, Verbose, "%f ms for MarkObjectsAsUnreachable Phase (%d Objects To Serialize)", ElapsedTime * 1000, InitialObjects.Num());
	}
}
```

#### Foundation - GarbageCollection - FRealtimeGC::MarkObjectAsUnreachabil(GarbageCollection.cpp)

``` cpp
FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	// EGatherOptions::None, EGatherOptions::Parallel
	EGatherOptions GatherOptions = GetObjectGatherOptions();

	// Don't swap the flags if we're re-entering this function to track garbage references
	if (const bool bInitialMark = !Stats.bFoundGarbageRef)
	{
		// This marks all UObjects as MaybeUnreachable
		FGCFlags::SwapReachableAndMaybeUnreachable();
	}
	else
	{
		// Swapping flags would inverse reachability results from the initial (normal) pass but what we want
		// is to reset reachability state of all objects to 'MaybeUnreachable'
		ResetReachabilityFlags(GatherOptions);
	}
	
	// Now make sure all clustered objects and root objects are marked as Reachable. 
	// This could be considered as initial part of reachability analysis and could be made incremental.
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
}
```

#### Foundation - GarbageCollection - FRealtimeGC::ResetReachabilityFlags(GarbageCollection.cpp)

``` cpp
void ResetReachabilityFlags(const EGatherOptions Options)
{
	// Options가 EGatherOptions::Parallel인 경우 병렬 처리를 위한 준비
	using FMarkObjectsState = TThreadedGather<TArray<UObject*>>;
	FMarkObjectsState MarkObjectsState;

	// 병렬 처리를 위한 쓰레드 모음
	MarkObjectsState.Start(Options, GUObjectArray.GetObjectArrayNum(), GUObjectArray.GetFirstGCIndex());

	// Reset all objects to 'MaybeUnreachable' state
	FMarkObjectsState::FThreadIterators& ThreadIterators = MarkObjectsState.GetThreadIterators();
	ParallelFor(TEXT("GC.ResetReachabilityFlags"), MarkObjectsState.NumWorkerThreads(), 1, [&ThreadIterators](int32 ThreadIndex)
		{
			FMarkObjectsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];
			for (; ThreadState.Index <= ThreadState.LastIndex; ++ThreadState.Index)
			{
				FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ThreadState.Index];
				if (ObjectItem->GetObject())
				{
					FGCFlags::SetMaybeUnreachable_ForGC(ObjectItem);
				}
			}
		}, (MarkObjectsState.NumWorkerThreads() == 1) ? EParallelForFlags::ForceSingleThread : EParallelForFlags::None);

	TArray<UObject*> Unused; // We didn't collect any objects when resetting so this is just a dummy to satisfy the API
	MarkObjectsState.Finish(Unused);

}
```

#### Foundation - GarbageCollection - FGCFlags::SetMaybeUnreachable_ForGC(GarbageCollectionInternalFlags.h)

``` cpp
FORCEINLINE static void SetMaybeUnreachable_ForGC(FUObjectItem* ObjectItem)
{
	// EInternalObjectFlags::ReachabililtyFlag0 삭제
	ObjectItem->AtomicallyClearFlag_ForGC(ReachableObjectFlag);
	// EInternalObjectFlags::ReachabilityFlag1 설정
	// 이 상태가 아직 GC 추적이 안된 상태? 이 상태를 유지하면 Purge 대상이 되는 듯?
	ObjectItem->AtomicallySetFlag_ForGC(MaybeUnreachableObjectFlag);
}
```

#### Foundation - GarbageCollection - FRealtimeGC::MarkRootObjectsAsReachable(GarbageCollection.cpp)

``` cpp
FORCENOINLINE void MarkRootObjectsAsReachable(const EGatherOptions Options, const EObjectFlags KeepFlags, TArray<UObject*>& OutRootObjects)
{
	{
		GRootsMutex.Lock();
		ProcessDirtyRootsNoLock();
		// RootSet을 Array로 복사
		TArray<int32> RootsArray(GRoots.Array());				
		GRootsMutex.Unlock();
		
		// 병렬로 작업 진행
		MarkRootsState.Start(Options, RootsArray.Num());
		FMarkRootsState::FThreadIterators& ThreadIterators = MarkRootsState.GetThreadIterators();
		
		// RootSet에 들어있는 UObject들을 Marking
		ParallelFor(TEXT("GC.MarkRootObjectsAsReachable"), MarkRootsState.NumWorkerThreads(), 1, [&ThreadIterators, &RootsArray](int32 ThreadIndex)
		{
				
			FMarkRootsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];

			while (ThreadState.Index <= ThreadState.LastIndex)
			{
				FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[RootsArray[ThreadState.Index++]];
				UObject* Object = static_cast<UObject*>(RootItem->GetObject());

				// IsValidLowLevel is extremely slow in this loop so only do it in debug
				checkSlow(Object->IsValidLowLevel());					

				FGCFlags::FastMarkAsReachableInterlocked_ForGC(RootItem);
				ThreadState.Payload.Add(Object);
			}
		}, (MarkRootsState.NumWorkerThreads() == 1) ? EParallelForFlags::ForceSingleThread : EParallelForFlags::None);	
	}
	
	using FMarkObjectsState = TThreadedGather<TArray<UObject*>>;
	FMarkObjectsState MarkObjectsState;
	
	// This is super slow as we need to look through all existing UObjects and access their memory to check EObjectFlags
	if (KeepFlags != RF_NoFlags)
	{
		MarkObjectsState.Start(Options, GUObjectArray.GetObjectArrayNum(), GUObjectArray.GetFirstGCIndex());

		FMarkObjectsState::FThreadIterators& ThreadIterators = MarkObjectsState.GetThreadIterators();
		// RootSet에 들어있지는 않지만 GC 대상에서 제외해야 할 Object 대상으로 Marking을 진행
		ParallelFor(TEXT("GC.SlowMarkObjectAsReachable"), MarkObjectsState.NumWorkerThreads(), 1, [&ThreadIterators, &KeepFlags](int32 ThreadIndex)
		{
			FMarkObjectsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];
			const bool bWithGarbageElimination = UObject::IsGarbageEliminationEnabled();

			while (ThreadState.Index <= ThreadState.LastIndex)
			{
				FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ThreadState.Index++];
				UObject* Object = static_cast<UObject*>(ObjectItem->GetObject());
				if (Object &&
					!ObjectItem->HasAnyFlags(EInternalObjectFlags_RootFlags) && // It may be counter intuitive to reject roots but these are tracked with GRoots and have already been marked and added
					!(bWithGarbageElimination && ObjectItem->IsGarbage()) && Object->HasAnyFlags(KeepFlags)) // Garbage elimination works regardless of KeepFlags
				{
					// IsValidLowLevel is extremely slow in this loop so only do it in debug
					checkSlow(Object->IsValidLowLevel());

					FGCFlags::FastMarkAsReachableInterlocked_ForGC(ObjectItem);
						ThreadState.Payload.Add(Object);
				}
			}
		}, (MarkObjectsState.NumWorkerThreads() == 1) ? EParallelForFlags::ForceSingleThread : EParallelForFlags::None);
	}

	// 결과를 OutRootObjects에 넣어서 반환
	MarkRootsState.Finish(OutRootObjects);
	MarkObjectsState.Finish(OutRootObjects);

}
```