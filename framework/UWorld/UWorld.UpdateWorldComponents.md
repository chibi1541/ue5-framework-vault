---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
  - "[[ULevel/ULevel.UpdateLevelComponents|ULevel::UpdateLevelComponents]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.GetSubsystemArrayCopy|FObjectSubsystemCollection::GetSubsystemArrayCopy]]"
  - "[[UWorldSubsystem/UWorldSubsystem.OnWorldComponentsUpdated|UWorldSubsystem::OnWorldComponentsUpdated]]"
tags:
  - World_cpp
---

```cpp
/** updates world components like e.g. line batcher and all level components */
void UpdateWorldComponents(bool bRerunConstructionScripts, bool bCurrentLevelOnly, FRegisterComponentContext* Context = nullptr)
{
    if (!IsRunningDedicatedServer())
    {
  		for (TObjectPtr<ULineBatchComponent>& LineBatcher : LineBatchers)
		{
			// haker:
	        // - ULineBatchComponent is herited from UPrimitiveComponent
	        // - ULineBatchComponent can be seen as UActorComponent
	        // - UWorld is NOT AActor, but has UActorComponent as its dynamic object for LineBatcher
	        // - LineBatchers should be registered separately
	        // 얘는 CDO에 들어가지 않는 오브젝트, 사실상 서브 오브젝트가 아님
			if (!LineBatcher)
			{
				// 여기 outer 지정도 없음
				LineBatcher = NewObject<ULineBatchComponent>();
				LineBatcher->bCalculateAccurateBounds = false;
			}
		}
	
		for (TObjectPtr<ULineBatchComponent>& LineBatcher : LineBatchers)
		{
			if (!LineBatcher->IsRegistered())
			{
				LineBatcher->RegisterComponentWithWorld(this, Context);
			}
		}
    }

    {
        for (int32 LevelIndex = 0; LevelIndex < Levels.Num(); ++LevelIndex)
        {
            ULevel* Level = Levels[LevelIndex];
            ULevelStreaming* StreamingLevel = FLevelUtils::FindStreamingLevel(Level);
            
            // update the level only if it is visible (or not a streamed level)
            if (!StreamingLevel || Level->bIsVisible)
            {
                Level->UpdateLevelComponents(bRerunConstructionScripts, Context);
                IStreamingManager::Get().AddLevel(Level);
            }
        }
    }

    const TArray<UWorldSubsystem*>& WorldSubsystems = SubsystemCollection.GetSubsystemArray<UWorldSubsystem>(UWorldSubsystem::StaticClass());
    for (UWorldSubsystem* WorldSubsystem : WorldSubsystems)
    {
        WorldSubsystem->OnWorldComponentsUpdated(*this);
    }
}
```

## 설명
- [[UWorld/UWorld.InitializeNewWorld|UWorld::InitializeNewWorld]]에서 `InitWorld` 이후에 호출되어, 월드에 속한 컴포넌트들을 등록(register)하는 함수.
- 크게 세 단계로 나뉜다: (1) `LineBatchers` 등록, (2) 각 `Levels`의 component 갱신, (3) `UWorldSubsystem`에 `OnWorldComponentsUpdated` 통지.
- `ULineBatchComponent`는 `UPrimitiveComponent` 계열이지만 `UWorld`가 직접 들고 있는 특수한 경우다. `UWorld`는 `AActor`가 아니므로 CDO에 포함되는 sub-object가 아니고, outer 지정도 없이 `NewObject`로 생성된 뒤 [[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]로 별도 등록된다.
- dedicated server에서는 렌더링이 없으므로 LineBatcher 관련 처리를 건너뛴다.
- 레벨 순회 시 streaming level이면서 `bIsVisible`이 false인 레벨은 제외한다. 즉 보이는 레벨(또는 streaming이 아닌 레벨)만 갱신한다.
- `FLevelUtils::FindStreamingLevel`로 스트리밍된 서브 레벨을 구분하는 과정은 복잡하므로, 일단 여기서는 "모든 서브 레벨에 대해 아래 로직을 수행한다" 정도로 인식하고 넘어간다.
- LineBatcher처럼 월드가 `UActorComponent`를 직접 들고 있는 것은 사실상 일반적이지 않은 상황이다. `AActor`는 `ULevel`에 종속되는 것이 일반적이므로, 실제 일반적인 초기화 경로는 [[ULevel/ULevel.UpdateLevelComponents|Level->UpdateLevelComponents()]] 쪽이다.
