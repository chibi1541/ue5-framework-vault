---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[AActor/AActor.RegisterAllActorTickFunctions|AActor::RegisterAllActorTickFunctions]]"
  - "[[AActor/AActor.RegisterActorTickFunctions|AActor::RegisterActorTickFunctions]]"
tags:
  - Actor_cpp
---

```cpp
/** thread safe container for actor related global variables */
// haker:
// - what is TLS(Thread-Local Storage)?
// - why is it thread-safe?
// - understand TLS with TThreadSingleton
// - VERY USEFUL concept to use parallel programming in client-side
// TThreadSingleton : TLS인 싱글톤
class FActorThreadContext : public TThreadSingleton<FActorThreadContext>
{
    friend TThreadSingleton<FActorThreadContext>;

    FActorThreadContext()
        : TestRegisterTickFunctions(nullptr)
    {}

    /** tests tick function registration */
    AActor* TestRegisterTickFunctions;
};
```

## 설명
- 액터 관련 전역 변수를 스레드 안전하게 담아두는 컨테이너다.
- `TThreadSingleton`은 TLS(Thread-Local Storage) 기반 싱글톤이다. 인스턴스가 스레드마다 하나씩 존재하므로, 전역처럼 접근하면서도 스레드 간 공유가 없어 lock 없이 안전하다. 클라이언트 측 병렬 프로그래밍에서 매우 유용한 패턴.
- 생성자가 private이고 `TThreadSingleton`을 friend로 두어, 오직 `Get()`을 통해서만 인스턴스를 얻게 강제한다.
- `TestRegisterTickFunctions`는 [[AActor/AActor.RegisterActorTickFunctions|RegisterActorTickFunctions()]]가 자신을 기록하고 [[AActor/AActor.RegisterAllActorTickFunctions|RegisterAllActorTickFunctions()]]가 `check()`로 검증/리셋하는 용도의 임시 변수다. virtual call chain이 최상위 구현까지 도달했는지 확인하는 디버깅 장치.
