---
related:
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.Initialize|FSubsystemCollectionBase::Initialize]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeValidatedSubsystem|FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.FindAndPopulateSubsystemArray|FSubsystemCollectionBase::FindAndPopulateSubsystemArray]]"
  - "[[FSubsystemCollectionInitialization/FSubsystemCollectionInitialization|FSubsystemCollectionInitialization]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection|FObjectSubsystemCollection]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.GetSubsystemArrayCopy|FSubsystemCollectionBase::GetSubsystemArrayCopy]]"
tags:
  - SubsystemCollection_h
---

```cpp
class FSubsystemCollectionBase
{
    // haker:
    // if we are dealing with UWorldSusystem:
    // - Outer is UWorld
    // - BaseType is UWorldSubsystem
    UObject* Outer;
    // 추후에 서브시스템의 인스턴스를 생성할 때 NewObject에 인자로 전달되는 ClassType
    UClass* BaseType;

		// 여기서 하나의 타입만 들어가도록 체크하니까
		// 오직 하니의 인스턴스만 생성 됨
    // haker: mapper from UClass(Subsystem's UClass) to UObject(Subsystem's instance)
    // - at initialization time of FObjectSubsystemCollection<SubsystemType>, one-to-one mappings are supported 
    TMap<TObjectPtr<UClass>, TObjectPtr<USubsystem>> SubsystemMap;

		// UDynamicSubsystem : 같은 Subsystem을 동적으로 하나 더 만들 수 있도록 하는 타입의 Subsystem
		// 존재 이유는? 모르겠당
    // haker: from my guess, to support UDynamicSubsystem, it persist multiple instances for each USubsystem's class type
    mutable TMap<UClass*, TArray<USubsystem*>> SubsystemArrayMap;
}
```

## 설명
- [[USubsystem/USubsystem|USubsystem]] 인스턴스들을 담는 컨테이너. `Outer`가 서브시스템들의 소유자(예: `UWorldSubsystem`이면 `UWorld`), `BaseType`이 인스턴스 생성 시 쓰이는 클래스 타입이다.
- `SubsystemMap`은 `UClass -> USubsystem` 1:1 매핑. 기본적으로 타입당 인스턴스 하나만 생성됨을 보장한다.
- `SubsystemArrayMap`은 `UDynamicSubsystem`처럼 같은 타입의 서브시스템을 여러 개 두는 경우를 지원하기 위한 것으로 추정.
