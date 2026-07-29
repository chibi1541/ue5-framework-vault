---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FSubsystemCollectionInitialization/FSubsystemCollectionInitialization|FSubsystemCollectionInitialization]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.Initialize|FSubsystemCollectionBase::Initialize]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeValidatedSubsystem|FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem]]"
tags:
  - SubsystemCollection_cpp
---

```cpp
void AddAndInitializeSubsystems(UClass* SubsystemClass)
{
	FSubsystemCollectionInitialization LocalInitialization;

	for (UClass* Class : SubsystemClasses)
	{
		// only add instances for non abstract Subsystems
    // haker:
    // - UClass::ClassFlags has class information
	  // 여기서 UWorldSubsystem(Subsystem들이 상속받는 상위 클래스)는 걸러짐
		if (Class->HasAllClassFlags(CLASS_Abstract) || Class->GetAuthoritativeClass() != Class)
		{
			continue;
		}
		
		// haker:
    // - ShouldCreateSubsystem() determines whether we create subsystem or not
    //    - override this method to control the flow of subsystem creation
    // - CDO is sufficient to call ShouldCreateSubsystem()
		if (!Class->GetDefaultObject<USubsystem>()->ShouldCreateSubsystem(Outer))
		{
			UE_LOGFMT(LogSubsystemCollection, Verbose, "Not creating subsystem of class {Class} as it returned false from ShouldCreateSubsystem",
				FTopLevelAssetPath(Class));
			continue;
		}
		LocalInitialization.ClassMap.Add(Class, Class);
		LocalInitialization.Queue.Add(Class);
	}

	// Store a map of classes we intend to initialize for re-entrant calls to GetSubsystem 
	for (UClass* ConcreteClass : SubsystemClasses)
	{
		// Only process classes which were added mapping to themselves above, not more classes which
		// were added during this loop
		if (LocalInitialization.ClassMap.FindRef(ConcreteClass) != ConcreteClass)
		{
			continue;
		}
		for (UClass* Parent = ConcreteClass->GetSuperClass();
			Parent != nullptr && Parent != BaseType && !LocalInitialization.ClassMap.Contains(Parent);
			Parent = Parent->GetSuperClass())
		{
			LocalInitialization.ClassMap.Add(Parent, ConcreteClass);
		}

		for (const FImplementedInterface& Interface : ConcreteClass->Interfaces)
		{
			if (!LocalInitialization.ClassMap.Contains(Interface.Class))
			{
				LocalInitialization.ClassMap.Add(Interface.Class, ConcreteClass);
			}
		}
	}

	// If we're re-entering initialization (e.g. loading a module during subsystem initialization) then add
	// our classes to the existing set/queue and allow the outer loop to do the creation
	if (Initialization)
	{
		for (const TPair<UClass*, UClass*>& Pair : LocalInitialization.ClassMap)
		{
			if (!Initialization->ClassMap.Contains(Pair.Key))
			{
				Initialization->ClassMap.Add(Pair.Key, Pair.Value);	
			}
		}
		Initialization->Queue.Append(LocalInitialization.Queue);
	}
	else 
	{
		// Only mark ourselves as populating now - some subsystems call GetSubsystem in their ShouldCreateSubsystem 
		TGuardValue PopulatingGuard(Initialization, &LocalInitialization);
		for (TPair<UClass*, UClass*> Pair : LocalInitialization.ClassMap)
		{
			// Only initialize the requested classes, other keys are for finding via the inheritence hierarchy 
			// with re-entrant calls to GetSubsystem
			// Skip creation if we created the subsystem re-entrantly
			if (Pair.Key == Pair.Value && !SubsystemMap.Contains(Pair.Key))
			{
				AddAndInitializeValidatedSubsystem(Pair.Key);
			}
		}
	}
}
```

## 설명
- [[FSubsystemCollectionBase/FSubsystemCollectionBase.Initialize|FSubsystemCollectionBase::Initialize]]에서 호출. 대상 클래스들 중 abstract가 아니고 `ShouldCreateSubsystem`이 true를 반환하는 클래스만 [[FSubsystemCollectionInitialization/FSubsystemCollectionInitialization|FSubsystemCollectionInitialization]](`LocalInitialization`)의 `ClassMap`/`Queue`에 등록한다.
- 상속 계층과 구현 인터페이스까지 `ClassMap`에 매핑해, 부모 클래스나 인터페이스로 `GetSubsystem`을 호출해도 concrete 클래스를 찾을 수 있게 한다.
- 재진입(다른 서브시스템 초기화 중 모듈 로딩 등으로 다시 들어온 경우)이면 기존 큐에 병합만 하고, 아니면 [[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeValidatedSubsystem|FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem]]로 실제 인스턴스를 생성한다.
