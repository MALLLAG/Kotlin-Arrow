# 리소스 (Resource)

## 개요

리소스의 할당과 해제는 쉽지 않습니다. 특히 서로 의존하는 여러 리소스가 있을 때 더욱 그렇습니다. **Resource DSL**은 리소스를 설치하고, 예외와 취소 상황에서도 적절한 종료 처리를 보장하는 기능을 제공합니다.

Arrow의 Resource는 **구조화된 동시성(Structured Concurrency)** 과 **KotlinX Coroutines**와 협력합니다.

> 리소스란 데이터베이스 연결, 파일 핸들, 네트워크 소켓 등 사용 후 반드시 정리(해제)해야 하는 것들을 말합니다. 정리하지 않으면 메모리 누수나 연결 풀 고갈 같은 문제가 발생합니다.

---

## 라이브러리 위치

| 라이브러리 | 설명 |
|-----------|------|
| `arrow-fx-coroutines` | 코루틴과 통합된 리소스 관리 |
| `arrow-autoclose` | 코루틴 없이 유사한 API 제공 |
```kotlin
// build.gradle.kts
dependencies {
    implementation("io.arrow-kt:arrow-fx-coroutines:버전")
}
```

---

## 문제 이해하기

### 안전하지 않은 코드 예시

다음 프로그램은 **안전하지 않습니다**. 서비스를 사용하는 동안 예외나 취소 신호가 발생하면 `dataSource`와 `userProcessor`가 누수될 수 있습니다.
```kotlin
class UserProcessor {
    fun start(): Unit = println("Creating UserProcessor")
    fun shutdown(): Unit = println("Shutting down UserProcessor")
}

class DataSource {
    fun connect(): Unit = println("Connecting dataSource")
    fun close(): Unit = println("Closed dataSource")
}

class Service(val db: DataSource, val userProcessor: UserProcessor) {
    suspend fun processData(): List<String> = 
        throw RuntimeException("I'm going to leak resources by not closing them")
}
```

### 리소스 누수가 발생하는 예
```kotlin
suspend fun example() {
    val userProcessor = UserProcessor().also { it.start() }
    val dataSource = DataSource().also { it.connect() }
    val service = Service(dataSource, userProcessor)
    
    service.processData()  // 여기서 예외 발생!
    
    // 아래 코드는 절대 실행되지 않음 → 리소스 누수!
    dataSource.close()
    userProcessor.shutdown()
}
```

---

## 기존 해결책의 한계 (use 함수)

Kotlin JVM을 사용한다면 `Closeable`이나 `AutoCloseable`에 의존하여 코드를 다시 작성할 수 있습니다:
```kotlin
suspend fun example() {
    UserProcessor().use { userProcessor ->
        userProcessor.start()
        DataSource().use { dataSource ->
            dataSource.connect()
            Service(dataSource, userProcessor).processData()
        }
    }
}
```

### use의 문제점

| 문제 | 설명 |
|------|------|
| **플랫폼 제한** | `Closeable`/`AutoCloseable`은 JVM 전용, Multiplatform에서 사용 불가 |
| **인터페이스 필수** | 인터페이스 구현 필요하거나 외부 타입을 래핑해야 함 |
| **중첩 콜백** | 여러 리소스는 콜백 트리로 중첩되어 합성이 어려움 |
| **메서드 이름 강제** | `close` 메서드 이름 강제 (`shutdown` 같은 이름 사용 불가) |
| **suspend 불가** | `fun close(): Unit`에서 suspend 함수 실행 불가 |
| **종료 신호 없음** | 성공, 에러, 취소 중 어떻게 종료되었는지 알 수 없음 |


---

## Resource의 핵심 개념

각 리소스는 **세 단계**를 가집니다:
```
┌─────────────────────────────────────────────────────────┐
│                 리소스 생명주기                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1️⃣ 획득 (Acquiring)                                    │
│     └── 리소스 생성 및 초기화                             │
│                                                         │
│  2️⃣ 사용 (Using)                                        │
│     └── 리소스로 작업 수행                                │
│                                                         │
│  3️⃣ 해제 (Releasing)                                    │
│     └── 리소스 정리 및 종료                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Resource**에서는 (1)과 (3) 단계를 묶어서 정의하고, 구현이 예외나 취소 상황에서도 모든 것이 올바르게 작동하도록 보장합니다.

> **Graceful Shutdown**: 애플리케이션이 종료될 때 리소스를 올바르게 해제하는 것은 여러 시나리오에서 중요합니다. [SuspendApp](https://arrow-kt.io/docs/suspendapp/)은 Resource를 개선하여 종료와 터미네이션을 우아하게 처리합니다.

---

## 리소스 제대로 다루기

Arrow의 Resource를 사용하는 두 가지 방법이 있습니다:

1. **`resourceScope 사용`**: `ResourceScope`를 리시버로 가진 함수들과 함께 사용
2. **`Resource<A> 값 사용`**: 전체 리소스 할당과 해제를 `Resource<A>` 값으로 래핑하여 나중에 더 큰 블록에서 사용

---

## 방법 1: resourceScope 사용

`ResourceScope` DSL을 사용하면 리소스를 설치하고 안전하게 상호작용할 수 있습니다.

### install 함수

배워야 할 유일한 연산은 `install`입니다:
```kotlin
install(
    acquire = { /* 리소스 획득 */ },
    release = { resource, exitCase -> /* 리소스 해제 */ }
)
```

- **acquire**: 리소스를 획득하는 코드
- **release**: 리소스를 해제하는 코드 (블록 끝에서 자동 실행 보장)
- **exitCase**: 실행이 어떻게 종료되었는지 (성공, 예외, 취소)

> **ExitCase의 종류**
>
> | ExitCase | 설명 |
> |----------|------|
> | `ExitCase.Completed` | 정상 완료 |
> | `ExitCase.Failure(e)` | 예외로 종료 |
> | `ExitCase.Cancelled` | 취소됨 |
>
> 이를 통해 종료 방식에 따라 다른 정리 작업을 수행할 수 있습니다!

### 예제
```kotlin
suspend fun ResourceScope.userProcessor(): UserProcessor =
    install(
        { UserProcessor().also { it.start() } }
    ) { p, _ -> p.shutdown() }

suspend fun ResourceScope.dataSource(): DataSource =
    install(
        { DataSource().also { it.connect() } }
    ) { ds, _ -> ds.close() }

suspend fun example(): Unit = resourceScope {
    val service = parZip(
        { userProcessor() },
        { dataSource() }
    ) { userProcessor, ds ->
        Service(ds, userProcessor)
    }
    val data = service.processData()
    println(data)
}
```

### 코드 분석
```
┌─────────────────────────────────────────────────────────┐
│  resourceScope 동작                                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  resourceScope {                                        │
│    ┌─ parZip으로 병렬 획득 ─┐                            │
│    │  userProcessor()     │  ← 동시에 시작               │
│    │  dataSource()        │  ← 동시에 연결               │
│    └──────────────────────┘                             │
│                                                         │
│    service.processData()  ← 예외 발생해도...          │
│                                                         │
│  } ← 여기서 자동으로 모든 리소스 해제!                     │
│      - dataSource.close()                               │
│      - userProcessor.shutdown()                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 취소 관련 중요 사항

> **NonCancellable 동작**
>
> `install`은 acquire와 release 단계를 `NonCancellable`로 호출합니다.
> - acquire 중 취소 신호나 예외 수신 시: 리소스가 획득되지 않은 것으로 간주되어 release 함수가 트리거되지 않음
> - 이미 획득된 다른 리소스들: 예상대로 해제 보장

---

## 방법 2: Resource<T> 값 사용

`resource` 사용법은 `install`과 매우 유사합니다. 주요 차이점은 결과가 `Resource<T>` 타입의 **값**이라는 것입니다.

### 핵심 차이점

| 특성 | install | resource |
|------|---------|----------|
| 반환 타입 | `T` (리소스 자체) | `Resource<T>` (레시피) |
| 실행 시점 | 즉시 | `.bind()` 호출 시 |
| 재사용성 | 불가 | 가능 (값으로 전달) |
```kotlin
val userProcessor: Resource<UserProcessor> = resource(
    { UserProcessor().also { it.start() } }
) { p, _ -> p.shutdown() }

val dataSource: Resource<DataSource> = resource(
    { DataSource().also { it.connect() } }
) { ds, exitCase ->
    println("Releasing $ds with exit: $exitCase")
    withContext(Dispatchers.IO) { ds.close() }
}

val service: Resource<Service> = resource {
    Service(dataSource.bind(), userProcessor.bind())
}

suspend fun example(): Unit = resourceScope {
    val data = service.bind().processData()
    println(data)
}
```

### Resource는 무엇인가?

> **왜 두 가지 방법을 제공하나요?**
>
> `resourceScope`가 일반적으로 더 좋은 문법을 제공하지만, 여러 리소스를 획득하는 것과 같은 사용 패턴은 단계들이 실제 클래스에 저장될 때 더 쉬워집니다.

실제로 `Resource`는 `ResourceScope`를 사용하는 매개변수 없는 함수의 타입 별칭일 뿐입니다:
```kotlin
typealias Resource<A> = suspend ResourceScope.() -> A
```

### 복잡한 시나리오를 위한 resource 블록

더 복잡한 시나리오를 위해 Arrow는 `ResourceScope`를 리시버로 가진 블록을 받는 `resource`를 제공합니다:
```kotlin
val userProcessor: Resource<UserProcessor> = resource {
    val x: UserProcessor = install(
        { UserProcessor().also { it.start() } },
        { processor, _ -> processor.shutdown() }
    )
    x
}
```

---

## Java와의 통합

JVM에서 실행 중이라면 Arrow는 `AutoCloseable`과의 내장 통합을 `closeable` 함수 형태로 제공합니다.
```kotlin
suspend fun example(): Unit = resourceScope {
    // AutoCloseable 자동 통합
    val inputStream = closeable { FileInputStream("file.txt") }
    val reader = closeable { BufferedReader(InputStreamReader(inputStream)) }
    
    // 사용...
    reader.readLine()
    
} // 자동으로 close() 호출
```

> **기존 Java 라이브러리 활용**: JDBC Connection, InputStream, OutputStream 등 Java의 모든 AutoCloseable 리소스를 쉽게 통합할 수 있습니다.

---

## Typed Errors와의 통합

리소스 관리는 typed error 빌더와 협력합니다. **스코프를 여는 순서가 동작에 영향을 미친다**는 것을 인식하는 것이 중요합니다.

### 패턴 1: either가 외부, resourceScope가 내부
```kotlin
either<String, Int> {
    resourceScope {
        val a = install({ }) { _, ex -> println("Closing A: $ex") }
        raise("Boom!")
    }
}
// 출력: Closing A: ExitCase.Cancelled
// 결과: Either.Left(Boom!)
```

`resourceScope`를 가로지르는 `bind`/`raise`는 release 종료자가 **`Cancelled`** 로 호출되게 합니다.
```
┌─────────────────────────────────────────────────────────┐
│  either (외부)                                           │
│  └── resourceScope (내부)                                │
│       └── raise("Boom!") → resourceScope 취소됨          │
│                                                         │
│  ExitCase: Cancelled (raise가 취소처럼 동작)              │
└─────────────────────────────────────────────────────────┘
```

### 패턴 2: resourceScope가 외부, either가 내부
```kotlin
resourceScope {
    either<String, Int> {
        val a = install({ }) { _, ex -> println("Closing A: $ex") }
        raise("Boom!")
    }
}
// 출력: Closing A: ExitCase.Completed
// 결과: (resourceScope 정상 완료)
```

`either`가 내부에 있으면 `raise`는 `either` 블록 내에서만 영향을 미치므로 리소스는 **정상 상태**로 해제됩니다.
```
┌─────────────────────────────────────────────────────────┐
│  resourceScope (외부)                                    │
│  └── either (내부)                                       │
│       └── raise("Boom!") → either만 종료                 │
│                                                         │
│  ExitCase: Completed (resourceScope는 정상 완료)         │
└─────────────────────────────────────────────────────────┘
```

### 핵심 포인트

> **두 경우 모두 리소스는 올바르게 해제됩니다.**
>
> 종료자가 모든 `ExitCase`에서 동일하게 작동한다면, 둘 사이에 눈에 보이는 차이가 없습니다.
>
> 하지만 `ExitCase`에 따라 다른 동작을 해야 한다면 (예: 로깅, 메트릭 수집), 중첩 순서가 중요합니다!

### 언제 어떤 순서를 사용할까?

| 시나리오 | 권장 순서 |
|----------|---------|
| raise를 "실패"로 취급하고 싶을 때 | `either { resourceScope { } }` |
| raise를 정상 흐름으로 취급하고 싶을 때 | `resourceScope { either { } }` |
| ExitCase에 따른 로깅이 필요할 때 | 상황에 맞게 선택 |

---

## 실용적 사용 사례

### 1. 데이터베이스 연결 풀
```kotlin
val connectionPool: Resource<HikariDataSource> = resource(
    { HikariDataSource(config).also { it.validate() } }
) { pool, exitCase ->
    when (exitCase) {
        is ExitCase.Completed -> println("정상 종료")
        is ExitCase.Failure -> println("에러로 종료: ${exitCase.failure}")
        is ExitCase.Cancelled -> println("취소됨")
    }
    pool.close()
}
```

### 2. HTTP 클라이언트
```kotlin
val httpClient: Resource<HttpClient> = resource(
    { HttpClient(CIO) { /* 설정 */ } }
) { client, _ -> client.close() }

suspend fun fetchData(): String = resourceScope {
    val client = httpClient.bind()
    client.get("https://api.example.com/data").bodyAsText()
}
```

### 3. 여러 의존성을 가진 서비스
```kotlin
suspend fun runApplication(): Unit = resourceScope {
    // 병렬로 리소스 획득
    val (db, cache, messageQueue) = parZip(
        { database.bind() },
        { redisCache.bind() },
        { rabbitMQ.bind() }
    ) { db, cache, mq -> Triple(db, cache, mq) }
    
    // 서비스 시작
    val service = Service(db, cache, messageQueue)
    service.run()
    
} // 모든 리소스 자동 해제 (역순)
```

### 4. ExitCase에 따른 다른 처리
```kotlin
val resource: Resource<Connection> = resource(
    { openConnection() }
) { conn, exitCase ->
    when (exitCase) {
        is ExitCase.Completed -> {
            conn.commit()
            conn.close()
        }
        is ExitCase.Failure -> {
            conn.rollback()
            conn.close()
            logger.error("Transaction failed", exitCase.failure)
        }
        is ExitCase.Cancelled -> {
            conn.rollback()
            conn.close()
            logger.warn("Transaction cancelled")
        }
    }
}
```
