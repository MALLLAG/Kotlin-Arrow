# 병렬 처리 (Parallelism)

## 개요

독립적인 계산들을 병렬로 수행하고 싶을 때가 자주 있습니다. 예를 들어, 데이터베이스에서 값을 가져오면서 동시에 다른 서비스에서 파일을 다운로드해야 한다면, 이것들을 동시에 수행하지 않을 이유가 없습니다.

---

## parZip - 고정된 수의 병렬 작업

`parZip` 을 사용하여 여러 계산의 실행을 결합할 수 있습니다.
```kotlin
suspend fun getUser(id: UserId): User = parZip(
    { getUserName(id) },
    { getAvatar(id) }
) { name, avatar -> 
    User(name, avatar) 
}
```

### parZip 구조 분석
```kotlin
parZip(
    { /* 첫 번째 병렬 작업 */ },
    { /* 두 번째 병렬 작업 */ },
    // ... 더 많은 작업들
) { result1, result2, /* ... */ ->
    // 모든 결과를 조합하는 마지막 블록
}
```

- **앞부분 인자들**: 병렬로 수행할 각 계산
- **마지막 블록 (trailing lambda)**: 모든 계산 결과를 받아서 최종 결과를 생성

> **왜 parZip이 중요한가?**
>
> `parZip`은 단순히 고수준의 동시성 뷰를 제공하는 것 이상입니다. 내부 구현에서 다음을 처리합니다:
> - **예외 전파**: 하나의 작업이 실패하면 적절히 예외를 전파
> - **취소 처리**: 하나의 작업이 실패하면 다른 실행 중인 작업들을 취소
>
> 이런 복잡한 작업을 직접 구현하면 버그가 발생하기 쉽습니다!

---

## parMap - 컬렉션에 대한 병렬 처리

계산이 컬렉션에 의존하는 경우, 예를 들어 모든 친구의 이름을 가져오고 싶다면 `parMap` 을 사용합니다.
```kotlin
suspend fun getFriendNames(id: UserId): List<User> =
    getFriendIds(id).parMap { getUserName(it) }
```

> **일반 map과의 차이**
> ```kotlin
> // 순차 처리 - 친구가 100명이면 100번 순차 호출
> friends.map { getUserName(it) }
> 
> // 병렬 처리 - 동시에 여러 요청 실행
> friends.parMap { getUserName(it) }
> ```

### 동시성 제한하기

컬렉션의 요소가 너무 많으면 과도한 동시성이 문제가 될 수 있습니다.
```kotlin
// 최대 10개의 작업만 동시에 실행
getFriendIds(id).parMap(concurrency = 10) { getUserName(it) }
```

> **주의**: 동시성을 제한하지 않으면 1000개의 요소에 대해 1000개의 코루틴이 동시에 실행될 수 있습니다. 이는 다음 문제를 야기할 수 있습니다:
> - 메모리 부족
> - 외부 API의 rate limit 초과
> - 데이터베이스 연결 풀 고갈

---

## Flow에서의 parMap

`parMap` 함수는 `Flow`에서도 제공됩니다.
```kotlin
flowOf(1, 2, 3, 4, 5)
    .parMap(concurrency = 3) { processItem(it) }
    .collect { println(it) }
```

### 동시성 팩터별 동작

| concurrency 값 | 동작 |
|---------------|------|
| 1 | 일반 `map`과 동일 (순차 처리) |
| 1 초과 | 내부 flow들이 동시에 수집됨 |

### parMapUnordered - 순서 무시로 성능 향상

출력 순서가 입력 순서와 같을 필요가 없다면 `parMapUnordered`를 사용하여 추가 성능을 얻을 수 있습니다.
```kotlin
// 순서 보장 - 먼저 시작해도 나중에 끝나면 기다림
flowOf(1, 2, 3).parMap { process(it) }

// 순서 무시 - 먼저 끝나는 대로 방출
flowOf(1, 2, 3).parMapUnordered { process(it) }
```

> **사용 사례**
> - `parMap`: 페이지네이션된 결과처럼 순서가 중요할 때
> - `parMapUnordered`: 독립적인 알림 발송처럼 순서가 상관없을 때

---

## awaitAll / parallelism (실험적 기능)

> **경고**: 이 섹션의 기능은 실험적입니다. 기본 개념은 유지되지만, API가 변경될 수 있습니다.

`parZip`이 높은 수준의 코드 뷰를 제공하지만, 특정 스타일로 코드를 작성해야 한다는 단점이 있습니다. Arrow는 일반적인 `async/.await()` 관용구를 사용하는 또 다른 도구를 제공합니다.
```kotlin
suspend fun getUser(id: UserId): User = awaitAll {
    val name = async { getUserName(id) }
    val avatar = async { getAvatar(id) }
    User(name.await(), avatar.await())
}
```

### awaitAll 동작 방식

- `async { }`: 비동기 계산 등록
- `.await()` 호출 시: 그 시점까지 등록된 **모든** async 계산들이 대기됨
- 예외 발생 시: 구조화된 동시성 규칙에 따라 전체 블록 취소

> **parZip vs awaitAll 비교**
>
> | 특성 | parZip | awaitAll |
> |------|--------|----------|
> | 스타일 | 함수형 | 명령형 |
> | 코드 구조 | 모든 작업을 인자로 전달 | 일반 코드처럼 작성 |
> | 적합한 상황 | 작업 수가 적고 명확할 때 | 복잡한 의존성이 있을 때 |
>
> `awaitAll` 내에서 독립적인 async 계산들의 시퀀스를 작성하는 것은 `parZip`에 인자로 전달하는 것과 동등합니다.

---

## Typed Errors와의 통합

Arrow의 typed errors는 Arrow Fx Coroutines 연산자들과 원활하게 통합되면서 구조화된 동시성 패턴을 지원합니다.

> **핵심 포인트**: DSL의 중첩 순서가 구조화된 동시성의 스코프 취소와 에러 처리에 영향을 미칩니다!

### 구조화된 동시성에서의 취소 이해하기

먼저, 취소가 어떻게 동작하는지 이해해야 합니다:
```kotlin
suspend fun logCancellation(): Unit = try {
    println("500 밀리초 동안 대기 중...")
    delay(500)
} catch (e: CancellationException) {
    println("대기가 조기에 취소되었습니다!")
    throw e  // CancellationException은 항상 다시 throw해야 함
}
```

> `CancellationException`은 코루틴이 취소되었음을 알리는 특별한 예외입니다. 이 예외는 반드시 다시 throw해야 합니다!

---

## 패턴 1: Raise DSL을 parZip 내부에 중첩
```kotlin
suspend fun example() {
    val triple = parZip(
        { either<String, Unit> { logCancellation() } },
        { either<String, Unit> { delay(100); raise("Error") } },
        { either<String, Unit> { logCancellation() } }
    ) { a, b, c -> Triple(a, b, c) }
    println(triple)
}
```

**출력:**
```
500 밀리초 동안 대기 중...
500 밀리초 동안 대기 중...
(Either.Right(kotlin.Unit), Either.Left(Error), Either.Right(kotlin.Unit))
```

### 동작 설명
```
parZip 스코프 (외부)
├── either 스코프 1 → Right(Unit) 반환
├── either 스코프 2 → Left("Error") 반환 (raise 발생)
└── either 스코프 3 → Right(Unit) 반환

결과: 모든 작업이 완료됨, 에러는 Either로 캡슐화됨
```

- **에러가 람다 내부에 머무름**: 다른 계산에 영향을 주지 않음
- `either` 내부의 `raise`는 해당 `either` 블록만 종료
- 다른 `parZip` 인자들은 계속 실행됨

> **사용 사례**: 각 작업의 성공/실패를 독립적으로 처리하고 싶을 때

---

## 패턴 2: parZip을 Raise DSL 내부에 중첩
```kotlin
suspend fun example() {
    val res = either {
        parZip(
            { logCancellation() },
            { delay(100); raise("Error") },
            { logCancellation() }
        ) { a, b, c -> Triple(a, b, c) }
    }
    println(res)
}
```

**출력:**
```
500 밀리초 동안 대기 중...
500 밀리초 동안 대기 중...
대기가 조기에 취소되었습니다!
대기가 조기에 취소되었습니다!
Either.Left(Error)
```

### 동작 설명
```
either 스코프 (외부)
└── parZip 스코프 (내부)
    ├── 작업 1: 대기 시작
    ├── 작업 2: 100ms 후 raise("Error") → 전체 parZip 취소 트리거
    └── 작업 3: 대기 시작
    
→ 작업 2의 raise로 인해 작업 1, 3 취소됨
→ 결과: Either.Left("Error")
```

- **에러가 구조화된 동시성에 의해 관찰됨**
- `raise`가 `CancellationException`처럼 동작하여 계산을 short-circuit
- 하나의 작업이 실패하면 다른 작업들이 **취소됨**

---

## parMap과 Raise DSL 결합

컬렉션 작업에서도 동일한 패턴을 적용할 수 있습니다:
```kotlin
suspend fun Raise<String>.failOnEven(i: Int): Unit {
    ensure(i % 2 != 0) { delay(100); "Error" }
    logCancellation()
}

suspend fun example() {
    val res = either {
        listOf(1, 2, 3, 4).parMap { failOnEven(it) }
    }
    println(res)
}
```

**출력:**
```
500 밀리초 동안 대기 중...
500 밀리초 동안 대기 중...
대기가 조기에 취소되었습니다!
대기가 조기에 취소되었습니다!
Either.Left(Error)
```

### 동작 분석

| 입력 | 짝수? | 결과 |
|-----|------|------|
| 1 | 아니오 | 대기 시작 → 취소됨 |
| 2 | 예 | `raise("Error")` → 전체 취소 트리거 |
| 3 | 아니오 | 대기 시작 → 취소됨 |
| 4 | 예 | `raise("Error")` |

- `failOnEven`은 짝수일 때 에러를 raise
- 입력 2와 4에서 실패 → 다른 코루틴들(1, 3) 취소됨

---

## 병렬 에러 누적 (parMapOrAccumulate)

모든 에러를 short-circuit하지 않고 **누적**하고 싶다면 `parMapOrAccumulate`를 사용합니다.
```kotlin
suspend fun example() {
    val res = listOf(1, 2, 3, 4)
        .parMapOrAccumulate { failOnEven(it) }
    println(res)
}
```

**출력:**
```
500 밀리초 동안 대기 중...
500 밀리초 동안 대기 중...
Either.Left(NonEmptyList(Error, Error))
```

### parMap vs parMapOrAccumulate 비교

| 특성 | parMap | parMapOrAccumulate |
|------|--------|-------------------|
| 하나가 실패하면 | 다른 작업들 취소 | 모든 작업 완료까지 대기 |
| 에러 결과 | 첫 번째 에러만 | 모든 에러 누적 (NonEmptyList) |
| 사용 사례 | 빠른 실패가 중요할 때 | 모든 에러를 수집해야 할 때 |

> **실용적 예시**
>
> **parMap 적합**: 결제 처리 - 하나라도 실패하면 전체 취소
> ```kotlin
> either {
>     orderItems.parMap { processPayment(it) }
> }
> ```
>
> **parMapOrAccumulate 적합**: 폼 유효성 검사 - 모든 에러 표시
> ```kotlin
> formFields.parMapOrAccumulate { validateField(it) }
> // 결과: "이름은 필수입니다", "이메일 형식이 올바르지 않습니다", ...
> ```

---

## Flow와 Raise DSL 사용 시 주의사항

> Typed errors를 KotlinX Flow와 함께 사용하면 raise DSL 스코프가 **누출**될 수 있으므로 주의해서 사용해야 합니다.
```kotlin
// ❌ 위험 - raise 스코프가 flow 외부로 누출될 수 있음
either {
    flowOf(1, 2, 3)
        .map { if (it == 2) raise("Error") else it }
        .collect { println(it) }
}

// ✅ 안전 - either를 flow 내부에서 사용
flowOf(1, 2, 3)
    .map { either { if (it == 2) raise("Error") else it } }
    .collect { println(it) }
```
