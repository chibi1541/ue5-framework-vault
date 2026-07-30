---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
tags:
  - TickTaskManager_cpp
---

```cpp
// haker: see the member-variable, 'Head'
struct FCoolingDownTickFunctionList
{
    bool Contains(FTickFunction* TickFunction) const
    {
        // haker: for junior, the data structure is important to understand the code
        FTickFunction* Node = Head;
        while (Node)
        {
            if (Node == TickFunction)
            {
                return true;
            }
            Node = Node->InternalData->Next;
        }
        return false;
    }

    // haker: we saw that InternalData->Next is the node for cooling-down list
    FTickFunction* Head;
};
```

## 설명
- cooling down 상태([[FTickFunction/FTickFunction|FTickFunction]]의 `ETickState::CoolingDown`)인 tick function들을 담는 linked list. [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]의 `AllCoolingDownTickFunctions`가 이 타입이다.
- 자기 자신은 `Head`만 들고 있고, 다음 노드는 각 tick function의 `FInternalData::Next`가 가리킨다. 즉 노드 저장 공간을 별도로 만들지 않는 intrusive linked list다.
- 그래서 `Contains()`도 `Head`부터 `InternalData->Next`를 따라가며 순회하는 형태다.
