---
related:
  - "[[UEngine/UEngine.Init|UEngine::Init]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]"
  - "[[Enum/EWorldType|EWorldType]]"
tags:
  - World_cpp
---

```cpp
using InitializationValues = FWorldInitializationValues;

// haker: note that this function is static function
// - when you look through parameters in a function, care about their types
// - when we use UWorld::CreateWorld(), we only pass two parameters, so rest of parameters is not necessary to care
static UWorld* CreateWorld(
    const EWorldType::Type InWorldType, 
    bool bInformEngineOfWorld, 
    FName WorldName = NAME_None, 
    UPackage* InWorldPackage = NULL, 
    bool bAddToRoot = true, 
    ERHIFeatureLevel::Type InFeatureLevel = ERHIFeatureLevel::Num, 
    // 월드를 만들 때 중요한 변수들을 모아놓은 플래그 변수
    const InitializationValues* InIVS = nullptr, 
    bool bInSkipInitWorld = false
)
{
    // haker: UPackage will be covered in the future, dealing with AsyncLoading
    // - for now, just think of it as describing file format for UWorld
    // - one thing to remember is that each world has 1:1 mapping on separate package
    //   - it is natural that all data in world needs to be serialized as file
    //   - in unreal engine, saving file means 'package', UPackage
    // - another thing is that UObject has OuterPrivate:
    //   - OuterPrivate infers where object is resides in
    //   - normally OuterPrivate is set as UPackage: 
    //     - where object is resides in == what file object resides in
    UPackage* WorldPackage = InWorldPackage;
    if (!WorldPackage)
    {
        // haker: UWorld needs package as its OuterPrivate and need to be serialized
        WorldPackage = CreatePackage(nullptr);
    }

    if (InWorldType == EWorldType::PIE)
    {
        // haker: like ObjectFlags in UObjectBase, UPackage's attribute can be set by flags in similar manner
        WorldPackage->SetPackageFlags(PKG_PlayInEditor);
    }

    // mark the package as containing a world
    // haker: what is 'Transient' in Unreal Engine?
    // - if you have property or asset which is not serialized, we mark it as 'Transient'
    // - Transient == (meta data to mark it as not-to-be-serialize)
    if (WorldPackage != GetTransientPackage())
    {
        WorldPackage->ThisContainsMap();
    }

    const FString WorldNameString = (WorldName != NAME_None) ? WorldName.ToString() : TEXT("Untitled");

    // haker: we set NewWorld's outer as WorldPackage:
    // - normally, when you look into outer object, it will finally end-up-with package(asset file containing this UObject)
    UWorld* NewWorld = NewObject<UWorld>(WorldPackage, *WorldNameString);

    // haker: UObject::SetFlags -> set unreal object's attribute with flag by bit operator (AND(&), OR(|), SHIFT(>>, <<) etc...)
    // - refer to EObjectFlags and UObject::ObjectFlags
    NewWorld->SetFlags(RF_Transactional);
    NewWorld->WorldType = InWorldType;
    NewWorld->SetFeatureLevel(InFeatureLevel);
    NewWorld->InitializeNewWorld(
        InIVS ? *InIVS : UWorld::InitializationValues()
            // haker: as we saw FWorldInitializationValues, the below member functions mark the flag to refer when we create new world
            .CreatePhysicsScene(InWorldType != EWorldType::Inactive)
            .ShouldSimulatePhysics(false)
            .EnableTraceCollision(true)
            .CreateNavigation(InWorldType == EWorldType::Editor)
            .CreateAISystem(InWorldType == EWorldType::Editor)
        , bInSkipInitWorld);
  
    // clear the dirty flags set during SpawnActor and UpdateLevelComponents
    WorldPackage->SetDirtyFlag(false);

    if (bAddToRoot)
    {
        // add to root set so it doesn't get GC'd
        NewWorld->AddToRoot();
    }

    // tell the engine we are adding a world (unless we are asked not to)
    if ((GEngine) && (bInformEngineOfWorld == true))
    {
        GEngine->WorldAdded(NewWorld);
    }

    return NewWorld;
}
```

## 설명
- `static` 함수이므로 인스턴스 없이 월드를 만들어낸다. 인자가 많지만 실제 호출부에서는 앞의 두 개(`InWorldType`, `bInformEngineOfWorld`)만 넘긴다.
- `UPackage`는 월드를 저장하는 하나의 asset 파일이라는 느낌. 직렬화를 위한 데이터이며 월드와 패키지는 1:1로 대응한다.
- `UObject`의 `OuterPrivate`는 그 객체가 어디에 존재하는지를 의미하고, 보통 `UPackage`로 설정된다. 여기서도 `NewObject<UWorld>(WorldPackage, ...)`로 outer를 패키지로 지정한다.
- `Transient`는 "직렬화하지 않겠다"는 의미의 meta data. transient package가 아닐 때만 `ThisContainsMap()`으로 맵 포함 표시를 한다.
- 월드 생성 옵션은 [[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]로 묶어서 `InitializeNewWorld`에 전달한다. navigation/AI는 `EWorldType::Editor`일 때만 생성한다.
- `bAddToRoot`로 root set에 추가해 GC 대상에서 제외하고, 마지막에 `GEngine->WorldAdded()`로 엔진에 월드 추가를 알린다.
