---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]"
tags:
  - SubsystemCollection_cpp
---

```cpp
USubsystem* FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem(UClass* SubsystemClass)
{
  // haker: create new USubsystem by SubsystemClass (in our case, the class is UWorldSubsystem)
  // Subsytem인 이유가 추상적이긴 하지만 이렇게 만들어진 애들의 Outer의 Subobject 개념이기 때문에
  // 그렇기 때문에 상위 outer가 살아 있는한 GC의 대상이 되지 않음
	USubsystem* Subsystem = NewObject<USubsystem>(Outer, SubsystemClass);
	SubsystemMap.Add(SubsystemClass,Subsystem);
	Subsystem->InternalOwningSubsystem = this;
	Subsystem->Initialize(*this);
	
	// Add this new subsystem to any existing maps of base classes to lists of subsystems
	// Not calling FatalErrorIfIteratingSubsystems because adding to the end of the array is safe for index-based iteration
	for (TPair<UClass*, TUniquePtr<FSubsystemArray>>& Pair : SubsystemArrayMap)
	{
		const bool bIsInterface = Pair.Key->IsChildOf<UInterface>();
		if ((!bIsInterface && SubsystemClass->IsChildOf(Pair.Key)) || 
			(bIsInterface && SubsystemClass->ImplementsInterface(Pair.Key)))
		{
			Pair.Value->Subsystems.Add(Subsystem);
		}
	}
}
```

## 설명
- [[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]에서 호출되어 실제로 [[USubsystem/USubsystem|USubsystem]] 인스턴스를 `NewObject`로 생성한다.
- `Outer`를 서브시스템의 Outer로 넘기기 때문에, 서브시스템은 Outer의 subobject 개념이 되어 Outer가 살아있는 한 GC 대상이 되지 않는다.
- 생성 후 `SubsystemMap`에 등록하고, `InternalOwningSubsystem`을 자기 자신(collection)으로 설정한 뒤 `Initialize`를 호출한다. 이미 존재하는 `SubsystemArrayMap` 캐시에도 상속/인터페이스 조건에 맞으면 추가한다.
