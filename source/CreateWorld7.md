---
summarize: true
---

#### Foundation - CreateWorld - AActor::GetComponents(Actor.h)

대표적인 GetComponents 함수

```cpp
/** 
 * get a direct reference to the components set rather than a copy with the null pointers removed 
 * WARNING: anything that could cause the component to change ownership or be destroyed will invalidate
 * this array, so use caution when iterating this set!
 */
const TSet<UActorComponent*>& GetComponents() const
{
    // haker: ObjectPtrDecal means removing ObjectPtr:
    // - OwnedComponents is TSet<TObjectPtr<UActorComponent>>
    // - Output is TSet<UActorComponent*>
    // TObjectPtr 뚜따 하고 싶은데 사용하는 함수
    return ObjectPtrDecay(OwnedComponents);
}

```

#### Foundation - CreateWorld - AActor::GetComponents(Actor.h)

```cpp
/**
 * UActorComponent specialization of GetComponents() to avoid unnecessaray casts
 * it is recommended to use TArrays with a TInlineAllocator to potentially avoid memory allocation costs
 * TInlineComponentArray is defined to make this easier, for example:
 * {
 *  TInlineComponentArray<UActorComponent*> PrimComponents;
 *  Actor->GetComponents(PrimComponents);
 * } 
 */
// haker: you can specify whether you include ChildActors or not
template <class AllocatorType>
void GetComponents(TArray<UActorComponent*, AllocatorType>& OutComponents, bool bIncludeFromChildActors = false)
{
    OutComponents.Reset();
    ForEachComponent_Internal<UActorComponent>(UActorComponent::StaticClass(), bIncludeFromChildActor, [&](UActorComponent* InComp)
    {
        OutComponents.Add(InComp);
    });
}
```

#### Foundation - CreateWorld - AActor::ForEachComponent_Internal(Actor.h)

```cpp
// haker: focus on how it uses ForEachComponent_Internal
// - bIsIncludeFromChildActor is dynamic variables (not static-compiled variable)
//   - so we call separate ForEachComponent_Internal with static-compiled variable
template <class ComponentType, typename Func>
void ForEachComponent_Internal(TSubclassOf<UActorComponent> ComponentClass, bool bIncludeFromChildActors, Func InFunc) const
{
		// 왜 ActorComponent인 경우를 나눴지?
    if (ComponentClass == UActorComponent::StaticClass())
    {
        if (bIncludeFromChildActors)
        {
		        // 템플릿 함수는 컴파일 타임에 인자 값이 확정되야 하기 때문에
		        // 아래와 같이 모든 버전을 일일히 정의함
            ForEachComponent_Internal<ComponentType, true /*bClassIsActorComponent*/, true /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
        else
        {
            ForEachComponent_Internal<ComponentType, true /*bClassIsActorComponent*/, false /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
    }
    else
    {
        if (bIncludeFromChildActors)
        {
            ForEachComponent_Internal<ComponentType, false /*bClassIsActorComponent*/, true /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
        else
        {
            ForEachComponent_Internal<ComponentType, false /*bClassIsActorComponent*/, false /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
    }
}
```

```cpp
/**
 * internal helper function to call a compile-time lambda on all components of a given-type
 * use template parameter bClassIsActorComponent to avoid doing unnecessary IsA checks when the ComponentClass is exactly UActorComponent
 * use template parameter bIncludeFromChildActors to recurse in to ChildActor components and find components of the appropriate type in those actors as well  
 */
template <class ComponentType, bool bClassIsActorComponent, bool bIncludeFromChildActors, typename Func>
void ForEachComponent_Internal(TSubclassOf<UActorComponent> ComponentClass, Func InFunc) const
{
    check(bClassIsActorComponent == false || ComponentClass == UActorComponent::StaticClass());
    check(ComponentClass->IsChildOf(ComponentType::StaticClass()));

		// 템플릿 함수가 컴파일 되는 타이밍에 인자 값이 정적으로 결정된 상태이므로
		// 템플릿 코드를 생성하는 과정에는 if문이 없어질 것임
    // static check, so that the most common case (bIncludeFromChildActors) doesn't need to allocate an additional array
    // haker: we are evaluating branch (condition) here:
    // - in compiled-time, it was fixed how the logic is flowed
    if (bIncludeFromChildActors)
    {
        TArray<AActor*, TInlineAllocator<NumInlinedActorComponents>> ChildActors;
        for (UActorComponent* OwnedComponent : OwnedComponents)
        {
            if (OwnedComponent)
            {
                // haker: by calling IsA(), we filter ActorComponent type
                if (bClassIsActorComponent || OwnedComponent->IsA(ComponentClass))
                {
                    InFunc(static_cast<ComponentType*>(OwnedComponent));
                }

                // haker: do you remember what ChildActorComponent is?
                // - for child-actor, we accumulate child-actor separately rather calling them in hierarchical manner
                if (UChildActorComponent* ChildActorComponent = Cast<UChildActorComponent>(OwnedComponent))
                {
                    if (AActor* ChildActor = ChildActorComponent->GetChildActor())
                    {
                        ChildActors.Add(ChildActor);
                    }
                }
            }
        }

        // haker: recursively call ForEachComponent_Internal() for each ChildActor
        for (AActor* ChildActor : ChildActors)
        {
            ChildActor->ForEachComponent_Internal<ComponentType, bClassIsActorComponent, bIncludeFromChildActors>(ComponentClass, InFunc);
        }
    }
    else
    {
        // haker: if we don't specify to include child-actor, just call OwnedComponents
        for (UActorComponent* OwnedComponent : OwnedComponents)
        {
            if (OwnedComponent)
            {
		            // 여기가 메인처리
		            // IsA 관계의 컴포넌트를 전부 찾아서 OutComponents로 반환
                if (bClassIsActorComponent || OwnedComponent->IsA(ComponentClass))
                {
                    InFunc(static_cast<ComponentType*>(OwnedComponent));
                }
            }
        }
    }
}
```

#### Foundation - CreateWorld - GetUnregisterdParent(Actor.cpp)

인자로 받은 Component와 같은 Owner(AActor)를 가지고 있고 bAutoRegister 설정이 true임에도 아직 Register되지 않은 가장 최상위 컴포넌트를 찾는 함수

```cpp
/**
 * walks through components hierarchy and returns closest to root parent component that is unregistered
 * only for components that belong to the same owner 
 */
// haker: return closest un-registered parent component:
// - constraints:
//   - AttachParent should have same owner(AActor) to Component
//   - AttachParent should be bAutoRegister as true
static USceneComponent* GetUnregisteredParent(UActorComponent* Component)
{
    USceneComponent* ParentComponent = nullptr;
    USceneComponent* SceneComponent = Cast<USceneComponent>(Component);

    while (SceneComponent
        // haker: we check all conditions for Component's AttachParent
        // - whether AttachParent's owner and Component's owner is same
        // - whether Component's AttachParent is registered
        //   --> this is final condition we exactly look for
        && SceneComponent->GetAttachParent()
        && SceneComponent->GetAttachParent()->GetOwner() == Component->GetOwner()
        && !SceneComponent->GetAttachParent()->IsRegistered())
    {
        SceneComponent = SceneComponent->GetAttachParent();
        if (SceneComponent->bAutoRegister && IsValidChecked(SceneComponent))
        {
            // we found unregistered parent that should be registered
            // but keep looking up the tree
            ParentComponent = SceneComponent;
        }
    }

    return ParentComponent;
}
```

#### Foundation - CreateWorld - class FObjectSubsystemCollection(SubsystemCollection.h)

```cpp
// World의 서브시스템 컬렉션 변수
FWorldSubsystemCollection SubsystemCollection;

class FWorldSubsystemCollection : public FObjectSubsystemCollection<UWorldSubsystem>
{
	
}

template<typename TBaseType>
class FObjectSubsystemCollection : public FSubsystemCollectionBase
{
		

}
```

#### Foundation - CreateWorld - FObjectSubsystemCollection::GetSubsystemArrayCopy(SubsystemCollection.h)

```cpp
/** Get a list of Subsystems by type */
template <typename TSubsystemClass>
TArray<TSubsystemClass*> GetSubsystemArrayCopy(const TSubclassOf<TSubsystemClass>& SubsystemClass) const
{
	// Force a compile time check that TSubsystemClass derives from TBaseType, the internal code only enforces it's a USubsystem
	TSubclassOf<TBaseType> SubsystemBaseClass = SubsystemClass;
	return FSubsystemCollectionBase::GetSubsystemArrayCopy<TSubsystemClass>(SubsystemBaseClass);
}
```

#### Foundation - CreateWorld - FSubsystemCollectionBase::GetSubsystemArrayCopy(SubsystemCollection.h)

```cpp
/** Get a list of Subsystems by type */
template<typename SubsystemType>
TArray<SubsystemType*> GetSubsystemArrayCopy(UClass* SubsystemClass) const
{
	if (const FSubsystemArray* Referenced = FindAndPopulateSubsystemArray(SubsystemClass))
	{
		return TArray<SubsystemType*>(reinterpret_cast<SubsystemType* const*>(Referenced->Subsystems.GetData()), Referenced->Subsystems.Num());
	}
	return {};
}
```

#### Foundation - CreateWorld - FSubsystemCollectionBase::FindAndPopulatesubsystemArray(SubsystemCollection.cpp)

```cpp
const FSubsystemCollectionBase::FSubsystemArray* FSubsystemCollectionBase::FindAndPopulateSubsystemArray(UClass* SubsystemClass) const
{
	// NOTE: There is no thread safety here for multiple threads trying to access a subsystem by a base class/interface.
	// Ideally there should be.
	const bool bIsInterface = SubsystemClass->IsChildOf<UInterface>();
	if (!SubsystemArrayMap.Contains(SubsystemClass))
	{
		FSubsystemArray NewList;
		for (auto Iter = SubsystemMap.CreateConstIterator(); Iter; ++Iter)
		{
			UClass* KeyClass = Iter.Key();
			if ((!bIsInterface && KeyClass->IsChildOf(SubsystemClass)) || 
				(bIsInterface && KeyClass->ImplementsInterface(SubsystemClass)))
			{
				NewList.Subsystems.Add(Iter.Value());
			}
		}
		if (NewList.Subsystems.Num())
		{
			return SubsystemArrayMap.Add(SubsystemClass, MakeUnique<FSubsystemArray>(MoveTemp(NewList))).Get();
		}
		// If nothing was found, we don't want to store this key in the map as we remove lists when they become empty.
		// We also don't pass the keys in SubsystemArrayMap to GC.
		return nullptr;
	}

	const TUniquePtr<FSubsystemArray>& List = SubsystemArrayMap.FindChecked(SubsystemClass);
	return List.Get();
}
```

#### Foundation - CreateWorld - UWorldSubsystem::OnWorldComponentsUpdated(WorldSubsystem.h)

WorldSubsystem 커스텀 시에 World Initialize 타이밍에 호출하고 싶은 로직은 아래 함수에 오버라이딩하는 목적

```cpp
/** called after world components (e.g. line batcher and all level components) have been updated */
virtual void OnWorldComponentsUpdated(UWorld& World) {}
```