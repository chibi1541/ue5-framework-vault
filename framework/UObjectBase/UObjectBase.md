---
related:
  - "[[UObjectBaseUtility/UObjectBaseUtility|UObjectBaseUtility]]"
  - "[[Enum/EObjectFlags|EObjectFlags]]"
tags:
  - UObjectBase_h
---

```cpp
/** low level implementation of UObject, should not be used directly in game code */
// haker: this class is the most base class for UObject
// - look through its member variables
class UObjectBase
{
    /**
     * Flags used to track and report various object states
     * this needs to be 8 byte aligned on 32-bit platforms to reduce memory waste 
     */
    // haker: bit flags to define UObject's behavior or attribute as meta-data format
    EObjectFlags ObjectFlags;

		// GetOuter()를 호출했을 때 나오는 녀석
		// UObject는 트리 구조로 구성됨
		// UPackage (패키지)
    //  └── UWorld (월드)
    //        └── ULevel (레벨)
    //                └── AActor (액터)
    //                        └── UActorComponent (컴포넌트)
    //                                └── UObject (서브오브젝트)
    /** object this object resides in */
    // haker: as we said previously, it is written as UPackage
    // - note that as times went by, the unreal supports lots of features to support reduce dependency on assets:
    //   - OFPA (One File Per Actor) is one of representative example
    //   - in the past, AActor resides in ULevel and its UPackage is just level asset file, which is straight-forward
    //   - but, after introducing OFPA, an indirection is added, no more each AActor is stored in ULevel file, it is stored in separate file Exteral path
    // - I'd like to say that overall pattern is maintained, but as engine evolves, it adds indirection and complexity to understand its actual behavior
    // - anyway for now, you just try to understand OuterPrivate will be set as UPackage normally, it is enought for now!
		ObjectPtr_Private::TNonAccessTrackedObjectPtr<UObject>						OuterPrivate;
};
```

## 설명
- `UObject`의 low level 구현부이자 최상위 base class. game code에서 직접 쓰지 않는다.
- `ObjectFlags`는 객체의 상태/속성을 meta data 형태로 표현하는 비트 플래그. [[Enum/EObjectFlags|EObjectFlags]] 참고.
- `OuterPrivate`은 `GetOuter()`로 얻는 값이며, 이 객체가 어디에 존재하는지를 나타낸다. UObject는 트리 구조를 이룬다: `UPackage` → `UWorld` → `ULevel` → `AActor` → `UActorComponent` → subobject.
- 보통 `OuterPrivate`은 `UPackage`로 설정된다. 다만 OFPA(One File Per Actor)를 쓰면 액터가 레벨에 종속되지 않고 별도 external 파일(하나의 UPackage)로 독립되어 indirection이 하나 추가된다.
