---
related:
  - "[[FRealtimeGC/FRealtimeGC.MarkObjectsAsUnreachable|FRealtimeGC::MarkObjectsAsUnreachable]]"
  - "[[Enum/EObjectFlags|EObjectFlags]]"
tags:
  - GarbageCollection_cpp
---

```cpp
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

## 설명
- 1단계: `GRootsMutex`로 보호된 root set(`GRoots`)을 배열로 복사한 뒤, `ParallelFor`로 각 root object를 `FastMarkAsReachableInterlocked_ForGC()`로 `Reachable` marking하고 결과에 추가한다.
- 2단계: `KeepFlags != RF_NoFlags`일 때만, root set에는 없지만 `KeepFlags`([[Enum/EObjectFlags|EObjectFlags]])를 가진 object를 찾아 `Reachable`로 marking한다.
  - 모든 UObject의 메모리에 접근해 flag를 확인해야 하므로 매우 느리다.
  - root flag를 가진 object는 1단계에서 이미 처리했으므로 제외한다.
  - garbage elimination이 켜져 있으면 `KeepFlags`와 무관하게 garbage object는 제외한다.
- 두 단계에서 모은 object를 `OutRootObjects`(`InitialObjects`)로 반환해 이후 참조 탐색의 시작점으로 사용한다.
