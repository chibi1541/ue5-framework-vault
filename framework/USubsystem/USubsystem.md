---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeValidatedSubsystem|FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]"
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
tags:
  - Subsystem_h
---

```cpp
class USubsystem : public UObject
{
    /**
     * override to control if the Subsystem should be created
     * for example you could only have your system created on servers
     * it is important to note that if using this is becomes very important to null check whenever getting the Subsystem
     * 
     * NOTE: this function is called on the CDO prior to instances being created!!! 
     */
    // UWorldSubsystem : Outer == UWorld
    // UGameInstanceSubsystem : Outer == UGameInstance
    // haker: as the comment describes, this member function is called via CDO(ClassDefaultObject)
    virtual bool ShouldCreateSubsystem(UObject* Outer) const { return true; }

    /** implement this for init/deinit of instances of the system */
    virtual void Initialize(FSubsystemCollectionBase& Collection) {}
    virtual void Deinitialize() {}

    // haker: we are interested UWorld's [FObjectSubsystemCollection<UWorldSubsystem>]
    // - each subsystem has its owner like this
    FSubsystemCollectionBase* InternalOwningSubsystem;
};
```

## 설명
- 특정 Object의 라이프사이클을 따라가는 자동 인스턴싱 매니저(싱글톤). `Engine`/`Editor`/`GameInstance`/`World`/`LocalPlayer` 단위로 지원되며, 각각 `UEngineSubsystem`/`UEditorSubsystem`/`UGameInstanceSubsystem`/`UWorldSubsystem`/`ULocalPlayerSubsystem`을 상속해서 만든다.
- `ShouldCreateSubsystem`은 CDO 시점에 호출되어 생성 여부를 결정한다(예: 서버에서만 생성).
- `Initialize`/`Deinitialize`만 재정의하면 나머지 라이프사이클 관리는 엔진이 대신 해준다.
- `InternalOwningSubsystem`이 이 서브시스템을 담고 있는 [[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]를 가리킨다.
