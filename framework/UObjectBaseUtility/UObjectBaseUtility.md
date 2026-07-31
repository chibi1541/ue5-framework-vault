---
related:
  - "[[UObject/UObject|UObject]]"
  - "[[UObjectBase/UObjectBase|UObjectBase]]"
  - "[[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|UObjectBaseUtility::IsTemplate]]"
tags:
  - UObjectBaseUtility_h
---

```cpp
/** provides utility function for UObject, this class should not be used directly */
// haker: later we'll cover UObjectBaseUtility's member functions below
class UObjectBaseUtility : public UObjectBase
{
    UObject* GetTypedOuter(UClass* Target) const
    {
        UObject* Result = NULL;
        for (UObject* NextOuter = GetOuter(); Result == NULL && NextOuter != NULL; NextOuter = NextOuter->GetOuter())
        {
            // haker: we are not getting into IsA(), which is out-of-scope cuz it is related to Reflection System
            if (NextOuter->IsA(Target))
            {
                Result = NextOuter;
            }
        }
        return Result;
    }

    /** traverses the outer chain searching for the next object of a certain type (T must be derived from UObject) */
    template <typename T>
    T* GetTypedOuter() const
    {
        return (T*)GetTypedOuter(T::StaticClass());
    }

    /** determine whether this object is a template object */
    // haker: 
    // - I have been look through the unreal engine source code for long time, but I still can't explain what is archetype object with specific example
    // - you just think of it as CDO, class default object
    //   - for CDO, what I understand is like **initialization list** as default UObject instance
    bool IsTemplate(EObjectFlags TemplateTypes = RF_ArchetypeObject|RF_ClassDefaultObject) const
    {
        // haker: note that if one of outer is template, the object is template
        for (const UObjectBaseUtility* TestOuter = this; TestOuter; TestOuter = TestOuter->GetOuter())
        {
            if (TestOuter->HasAnyFlags(TemplateTypes))
                return true;
        }
        return false;
    }
```

## 설명
- `UObject`를 위한 utility 함수를 제공하는 클래스로, 직접 사용하는 클래스는 아니다.
- `GetTypedOuter()`는 outer chain을 따라 올라가며 원하는 타입의 outer를 찾는다. 템플릿 버전은 `T::StaticClass()`를 넘겨주는 래퍼.
- `IsTemplate()`은 archetype object 또는 CDO(class default object)인지 판별한다. 자기 자신뿐 아니라 outer 중 하나라도 template이면 template으로 취급한다.
- CDO는 default `UObject` 인스턴스를 위한 initialization list 같은 것으로 이해하면 된다.
