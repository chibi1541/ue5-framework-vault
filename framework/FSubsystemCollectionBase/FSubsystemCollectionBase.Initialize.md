---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[UWorld/UWorld.InitializeSubsystems|UWorld::InitializeSubsystems]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]"
tags:
  - SubsystemCollection_cpp
---

```cpp
/** initialize the collection of systems; systems will be created and initialized */
void Initialize(UObject* NewOuter)
{
    // already initialized
    if (Outer)
    {
        return;
    }

    // haker: we set NewOuter as UWorld
    Outer = NewOuter;
    // BaseType은 생성자에서 이미 할당 됨
    
    ****if (ensure(BaseType) && ensure(SubSystemMap.Num() == 0))
    {
	      check(IsInGameThread());

				// 예전에는 UDynamicSubsystem 이랑 Uubsystem를 따로 구분했는데 요즘은 섞어 놓는듯?
        // haker: BaseType is UWorldSubsystem for UWorld::SubsystemCollection
        // - calling GetDerivedClasses() collects all classes derived from UWorldSubsystem
        TArray<UClass*> SubsystemClasses;
        
        // haker: I eliminate UDynamicsSubsystem handling codes for simplicity
        // - the below code is about non-UDynamicSubsystem e.g. UWorldSubsystem
        if (BaseType->IsChildOf(UDynamicSubsystem::StaticClass()))
				{
					for (const TPair<FName, TArray<UClass*>>& ModuleClasses : GlobalDynamicSystemModuleMap)
					{
						for (UClass* SubsystemClass : ModuleClasses.Value)
						{
							if (SubsystemClass->IsChildOf(BaseType))
							{
								SubsystemClasses.Add(SubsystemClass);
							}
						}
					}
				}
				else
				{
					// 여기서 클래스 정보만 수집
          GetDerivedClasses(BaseType, SubsystemClasses, true);
				}
          
				AddAndInitializeSubsystem(SubsystemClass);

        // haker: see the definition of GlobalSubsystemCollections
        // Subsystem의 생성을 글로벌하게 체크
        GlobalSubsystemCollections.Add(this);
    }
}
```

## 설명
- [[UWorld/UWorld.InitializeSubsystems|UWorld::InitializeSubsystems]]에서 호출. `Outer`(`FSubsystemCollectionBase`의 기존 멤버)가 이미 설정되어 있으면 중복 초기화를 막는다.
- `BaseType`을 상속하는 모든 `UClass`(`SubsystemClasses`)를 `GetDerivedClasses`로 수집한 뒤 [[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]에 넘겨 실제 인스턴스를 생성/초기화한다.
- `UDynamicSubsystem` 분기는 예전에 별도로 다뤘던 것으로 보이나 최근에는 일반 흐름과 섞여 처리되는 듯.
