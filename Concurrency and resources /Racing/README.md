# 레이싱 (Racing)

## 개요

병렬 처리 연산자들은 수행하는 **모든** 계산의 결과에 관심이 있는 경우를 다룹니다. 하지만 파일을 다운로드할 때 복원력(resilience)을 위해 두 서버에서 동시에 시도하는 시나리오를 상상해 보세요. 한 서버에서 파일을 받으면, 나머지에는 더 이상 관심이 없습니다.

이것이 바로 두 계산을 **레이싱**하는 예시입니다.

> 레이싱은 여러 작업을 동시에 시작하고 **가장 먼저 성공한 결과만** 취하는 패턴입니다.

---

## 레이싱의 핵심 원칙

레이싱의 핵심은 **첫 번째로 성공한 값만 신경 쓴다**는 것입니다. 구체적인 동작은 다음과 같아야 합니다:

1. **레이서가 생성한 첫 번째 값이 레이스에서 승리**
2. **모든 예외는 로그로 기록되지만 레이스에서 이기지 못함** - 레이스가 끝날 때까지 대기해야 함
3. **레이스가 끝나면 승리 값을 반환하기 전에 모든 레이서가 취소됨** - 이는 획득한 모든 리소스가 닫히면서도 성공 값을 최대한 빨리 반환하는 것을 보장
```
┌─────────────────────────────────────────────────────┐
│                    Racing 동작                       │
├─────────────────────────────────────────────────────┤
│  서버1 ──────────────× (실패, 대기)                    │
│  서버2 ─────────✓ (성공!) → 결과 반환                   │
│  서버3 ─────────────────× (취소됨)                     │
│                    ↑                                │
│              승리 시점에서                             │
│           다른 모든 레이서 취소                          │
└─────────────────────────────────────────────────────┘
```

---

## 간단한 레이싱 (raceN)

간단한 경우를 위해 Arrow는 2개 또는 3개의 계산에 대해 레이싱을 수행하는 함수를 제공합니다.
```kotlin
suspend fun file(server1: String, server2: String) =
    raceN(
        { downloadFrom(server1) },
        { downloadFrom(server2) }
    ).merge()
```

### 코드 분석

| 부분 | 설명 |
|------|------|
| `raceN` | 2~3개의 계산을 레이싱 |
| `{ downloadFrom(server1) }` | 첫 번째 레이서 |
| `{ downloadFrom(server2) }` | 두 번째 레이서 |
| `.merge()` | `Either<A, B>`를 단일 값으로 병합 |

> **merge()가 필요한 이유**
>
> `raceN`의 결과는 `Either<A, B>`입니다. 각 타입은 `raceN`의 각 분기에 해당합니다.
> 두 계산이 같은 타입을 반환하고 어느 것이 "이겼는지" 상관없다면, `.merge()`로 단일 값으로 합칩니다.
>
> ```kotlin
> // raceN 결과: Either<File, File>
> // merge() 후: File
> ```

---

## select를 사용한 구현의 복잡성

코루틴 표준 라이브러리는 `select` 표현식을 제공하지만, 종종 더 높은 수준의 DSL이 필요합니다. 예를 들어, 첫 번째 성공 값을 위해 레이싱하고 나머지를 취소할 때, 또는 typed errors와 쉽게 결합하고 싶을 때 말이죠.

먼저 `select`로 이 예제를 어떻게 작성하는지 보고, Arrow Fx Coroutines로 어떻게 다시 작성할 수 있는지 봅시다.

### select를 사용한 복잡한 구현
```kotlin
object RemoteCache {
    suspend fun getUser(id: UserId): User =
        if (Random.nextBoolean()) User("$id-remote-user")
        else throw BadRequestException()
}

object LocalCache {
    suspend fun getUser(id: UserId): User =
        if (Random.nextBoolean()) User("$id-local-user")
        else throw NullPointerException()
}

// 에러 발생 후 취소될 때까지 대기하는 헬퍼 함수
suspend fun <A> awaitAfterError(block: suspend () -> A): A = try {
    block()
} catch (e: Throwable) {
    if (e is CancellationException || NonFatal(e)) throw e
    e.printStackTrace()
    awaitCancellation()
}

suspend fun getRemoteUser(id: UserId): User = coroutineScope {
    try {
        select {
            async { awaitAfterError { RemoteCache.getUser(id) } }.onAwait { it }
            async { awaitAfterError { LocalCache.getUser(id) } }.onAwait { it }
        }
    } finally {
        coroutineContext.job.cancelChildren()
    }
}
```

### 처리해야 하는 엣지 케이스들

위 코드는 네 가지 중요한 엣지 케이스를 처리합니다:

| 요소 | 목적 |
|------|------|
| `coroutineScope { }` | 반환 전에 모든 활성 코루틴이 완료되도록 보장 |
| `try/finally` | select가 반환되면 아직 실행 중인 코루틴들을 **항상 취소** (대기하지 않음). 이것이 없으면 모든 값을 기다리게 되어 레이싱의 목적이 무효화됨 |
| `awaitAfterError` | 레이싱 중 참가자에게 에러가 발생하면, 그들은 "패배자"로 간주되어 레이스가 끝날 때까지 대기해야 함. 승자가 레이스를 취소하므로 `awaitCancellation`으로 구현 |
| `e.printStackTrace()` | 레이싱 중 에러가 발생한 참가자는 패배자이지만, 이 정보가 사라지는 것은 바람직하지 않으므로 에러 처리 전략이 필요 |
| `CancellationException` 또는 치명적 예외 체크 | `CancellationException`이나 `OutOfMemoryException` 같은 치명적 예외에서는 절대 복구하면 안 됨 |

> 이 설정 과정은 저수준이고 복잡합니다. 버그가 발생하기 쉽습니다!

---

## Racing DSL (실험적 기능)

> 이 섹션의 기능은 실험적입니다. 기본 개념은 유지되지만, API가 변경될 수 있습니다.

Arrow는 `select` 위에 더 간단한 레이싱 DSL을 제공하여 위의 복잡한 의미론을 보장합니다.
```kotlin
suspend fun getUserRacing(id: UserId): User = racing {
    race { RemoteCache.getUser(id) }
    race { LocalCache.getUser(id) }
}
```

> **주의**: 모든 레이서가 예외를 던지면 `racing` 블록은 성공 값을 기다리며 **영원히 멈춥니다**.

---

## 타임아웃 처리

모든 것이 실패하면 레이싱에서 멈춤(hang)이 흔합니다. 따라서 타임아웃을 포함하는 것이 일반적입니다.
```kotlin
suspend fun getUserRacing(id: UserId): User = racing {
    race { RemoteCache.getUser(id) }
    race { LocalCache.getUser(id) }
    race { 
        delay(10.milliseconds)
        throw TimeoutException()
    }
}
```

### 타임아웃 패턴 변형
```kotlin
// 패턴 1: 예외 던지기
race {
    delay(10.milliseconds)
    throw TimeoutException()
}

// 패턴 2: 기본값 반환
race {
    delay(10.milliseconds)
    User.DEFAULT
}

// 패턴 3: null 반환 (nullable 반환 타입일 때)
race {
    delay(10.milliseconds)
    null
}

// 패턴 4: Either 사용
race {
    delay(10.milliseconds)
    Either.Left(TimeoutError)
}

// 패턴 5: raise 사용 (typed errors)
race {
    delay(10.milliseconds)
    raise(TimeoutError)
}
```

---

## 예외가 이길 수 있게 허용하기 (raceOrFail)

때로는 모든 예외를 무시하는 것이 바람직하지 않습니다. 특정 참가자의 예외가 레이스에서 이기도록 허용해야 할 때가 있습니다.

### 사용 시나리오

- `LocalCache`: 캐시 히트 시 빠르게 성공, 캐시 미스 시 느림
- `RemoteCache`: `BadRequest`나 `Unauthorized`로 레이스를 더 빨리 끝낼 수 있음
- 단, `HttpClient`는 성공적인 캐시 히트 전에 `ConnectException`으로 실패할 수도 있음
```kotlin
suspend fun getUserRacing(id: UserId): User = racing {
    raceOrFail { RemoteCache.getUser(id) }  // 예외도 레이스 결과가 될 수 있음
    race { LocalCache.getUser(id) }          // 예외는 무시됨
}
```

### race vs raceOrFail 비교

| 함수 | 성공 시 | 예외 발생 시 |
|------|--------|------------|
| `race { }` | 레이스 승리 | 대기 (다른 레이서에게 기회 양보) |
| `raceOrFail { }` | 레이스 승리 | **예외도 레이스 승리** (전파됨) |
```
┌─────────────────────────────────────────────────────────┐
│  raceOrFail 동작                                         │
├─────────────────────────────────────────────────────────┤
│  Remote (raceOrFail) ──× BadRequest → 예외 전파! 레이스 종료 │
│  Local (race) ─────────────────── (취소됨)                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  race 동작                                               │
├─────────────────────────────────────────────────────────┤
│  Remote (race) ──× BadRequest → 대기...                  │
│  Local (race) ─────────────✓ 성공! → 결과 반환            │
└─────────────────────────────────────────────────────────┘
```

---

## 특정 조건까지 레이싱하기

때로는 값이 생성될 때까지만 레이싱하는 것이 아니라, **특정 조건을 만족해야** 할 때가 있습니다.

### 예제: NonEmptyList 가져오기
```kotlin
suspend fun LocalCache.getCachedUsers(ids: NonEmptyList<UserId>): List<User> =
    ids.mapNotNull { id -> getUserOrNull(id) }

suspend fun getUserRacing(ids: NonEmptyList<UserId>): List<User> = racing {
    race { RemoteCache.getUsers(ids) }
    race(condition = { it.size == ids.size }) {
        LocalCache.getCachedUsers(ids)
    }
}
```

### 동작 설명

`getCachedUsers`는 캐시에 없는 값을 무시하므로, 캐시의 결과가 요청된 id 수와 일치하는지 확인해야 합니다.
```kotlin
// 요청: ids = [1, 2, 3] (3개)
// 
// RemoteCache: [User1, User2, User3] → 조건 없음, 성공하면 승리
// LocalCache: [User1, User3] → 2개만 반환, 조건 불충족 → 대기
// LocalCache: [User1, User2, User3] → 3개 반환, 조건 충족 → 승리 가능
```

> **주의**: 모든 레이서가 실패하거나 조건을 충족하지 못하면 `racing` 블록은 성공 값을 기다리며 **영원히 멈춥니다**.

---

## 커스텀 예외 처리

기본적으로 처리되지 않은 예외에 대한 전략은 `Throwable::printStackTrace`이지만, 설치된 `CoroutineExceptionHandler`가 있으면 그것을 사용하려고 시도합니다.

예를 들어 Ktor의 경우 Ktor의 `defaultCoroutineExceptionHandler`를 사용합니다.

### 커스텀 핸들러 설치

`withContext`를 사용하여 `CoroutineExceptionHandler`를 명시적으로 설치하면 쉽게 재정의할 수 있습니다.
```kotlin
suspend fun customErrorHandling(): String =
    withContext(CoroutineExceptionHandler { ctx, t ->
        // 커스텀 로깅
        logger.error("Racing error", t)
        // 또는 모니터링 시스템에 보고
        monitoring.reportError(t)
    }) {
        racing {
            race {
                delay(2.seconds)
                throw RuntimeException("boom!")
            }
            race {
                delay(10.seconds)
                "Winner!"
            }
        }
    }
```

---

## Typed Errors와의 통합

레이싱 목적상, `raise`를 사용하는 것은 **예외를 던지는 것과 동일한 효과**를 가집니다.
```kotlin
// raise는 예외와 동일하게 취급됨
race {
    raise(UserNotFound)  // 이 레이서는 "패배"하고 대기
}

// 에러를 전파하려면 raceOrFail 사용
raceOrFail {
    raise(UserNotFound)  // 이 에러가 레이스 결과가 됨
}
```

### 패턴 정리

| 목적 | 사용할 함수 |
|------|-----------|
| typed error가 레이스 결과가 되어야 함 | `raceOrFail` |
| typed error를 성공으로 취급하지 않음 | `race` |
```kotlin
// 예제: either와 racing 결합
suspend fun getUserSafely(id: UserId): Either<UserError, User> = either {
    racing {
        raceOrFail { 
            // raise가 전파되어 Either.Left가 됨
            RemoteCache.getUser(id) 
        }
        race { 
            // raise는 무시되고 다른 레이서에게 기회 양보
            LocalCache.getUser(id) 
        }
    }
}
```

---

## 실용적 사용 사례

### 1. 다중 CDN 파일 다운로드
```kotlin
suspend fun downloadFile(fileId: String): ByteArray = racing {
    race { cdn1.download(fileId) }
    race { cdn2.download(fileId) }
    race { cdn3.download(fileId) }
    race {
        delay(30.seconds)
        throw DownloadTimeoutException()
    }
}
```

### 2. 캐시 우선 전략 with 폴백
```kotlin
suspend fun getData(key: String): Data = racing {
    race { localCache.get(key) }           // 가장 빠를 가능성
    race { distributedCache.get(key) }     // 중간 속도
    raceOrFail { database.get(key) }       // 가장 느리지만 신뢰할 수 있음
}
```

### 3. 헬스체크
```kotlin
suspend fun healthCheck(): HealthStatus = racing {
    race { 
        checkAllServices()
        HealthStatus.HEALTHY
    }
    race {
        delay(5.seconds)
        HealthStatus.DEGRADED  // 타임아웃 시 성능 저하 상태
    }
}
```
