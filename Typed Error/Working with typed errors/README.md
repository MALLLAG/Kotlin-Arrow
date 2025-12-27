# 타입화된 에러(Typed Errors) 다루기

**타입화된 에러**는 함수형 프로그래밍의 기법으로, 코드 실행 중 발생할 수 있는 잠재적 에러를 시그니처(또는 타입)에 명시적으로 표현합니다. 이는 예외(exception)를 사용할 때와 다릅니다. 예외에서는 어떤 문서를 제공하더라도 컴파일러가 고려하지 않아, 방어적인 에러 처리 모드로 이어집니다.

---

## 핵심 개념과 타입

### 논리적 실패(Logical Failure) vs 실제 예외(Real Exception)

| 구분 | 논리적 실패 | 실제 예외 |
|------|------------|----------|
| 정의 | 도메인에서 성공으로 간주되지 않지만, 여전히 도메인 내에 있는 상황 | 주로 기술적인 문제로, 진정으로 예외적이며 도메인의 일부가 아닌 것 |
| 예시 | 사용자 저장소에서 특정 쿼리에 대한 사용자를 찾지 못함 | 데이터베이스 연결 끊김, 네트워크 타임아웃, 호스트 불가 |
| 처리 방법 | 타입화된 에러로 표현 | Arrow의 복원력(resilience) 메커니즘 사용 |

> 💡 **실제 개발에서의 구분 예시**:
> ```kotlin
> // 논리적 실패 - 도메인 로직의 일부
> sealed class UserError {
>     object NotFound : UserError()           // 사용자 없음
>     object InvalidEmail : UserError()       // 이메일 형식 오류
>     object PasswordTooWeak : UserError()    // 비밀번호 약함
> }
> 
> // 실제 예외 - 도메인 외부의 문제
> // DatabaseException, NetworkTimeoutException, OutOfMemoryError
> ```

### 성공(Success)과 실패(Failure)

에러 처리에서 우리는 종종 **성공(happy path)** 과 **실패**를 구분합니다:
- **성공**: 모든 것이 의도대로 작동하는 경우
- **실패**: 문제가 발생한 경우

접근 방식에 따라 함수의 시그니처는 실패 가능성만 신호하거나, 발생할 수 있는 문제의 범위를 추가로 설명합니다.

---

## 두 가지 접근 방식

함수의 시그니처에서 에러를 표현하는 두 가지 주요 접근 방식이 있습니다. Arrow는 모든 방식에 대해 **통일된 API**를 제공합니다.

### 1. 래퍼 타입(Wrapper Type) 접근법

함수의 반환 타입이 에러 선택을 제공하는 더 큰 타입 내에 중첩됩니다. 이렇게 하면 **에러가 값으로 표현**됩니다.
```kotlin
fun findUser(id: UserId): Either<UserNotFound, User>
```

이 시그니처는 `findUser`의 결과가 성공 시 `User` 타입이고, 논리적 실패 발생 시 `UserNotFound`임을 표현합니다.

**주요 래퍼 타입**:
- Kotlin 표준 라이브러리: 포함할 수 있는 정보가 제한됨
- Arrow: `Either`, `Ior` - 개발자가 논리적 실패의 타입을 선택할 수 있고, 첫 번째 타입 파라미터로 반영

### 2. 연산 컨텍스트(Computation Context) 접근법

논리적 실패로 끝날 수 있는 능력이 `Raise<E>`가 함수의 컨텍스트나 스코프의 일부가 되는 것으로 표현됩니다.
```kotlin
// Raise<UserNotFound>가 확장 리시버인 경우
fun Raise<UserNotFound>.findUser(id: UserId): User

// Raise<UserNotFound>가 컨텍스트 파라미터인 경우 (미래)
context(_: Raise<UserNotFound>) 
fun findUser(id: UserId): User
```

> 💡 **어떤 것을 선택해야 할까?**
>
> | 래퍼 타입 (`Either`) | 연산 컨텍스트 (`Raise`) |
> |---------------------|----------------------|
> | 값으로 에러를 다룸 | 컨텍스트로 에러를 다룸 |
> | `Either<E, A>` 반환 | 직접 `A` 반환 |
> | Java 호환성 좋음 | 더 깔끔한 시그니처 |
> | 익숙한 패턴 | 최신 Kotlin 스타일 |
>
> 두 방식은 상호 운용 가능하므로, 프로젝트에 맞는 방식을 선택하세요!

---

## 성공 / Happy Path 정의하기

### 래퍼 타입 접근법
```kotlin
object UserNotFound
data class User(val id: Long)

// 값을 생성할 때는 성공을 나타내는 타입으로 래핑 필요
val user: Either<UserNotFound, User> = User(1).right()
```

`Either`에서 값(val)을 생성하므로, 성공을 나타내는 타입으로 추가 래핑해야 합니다. 이 타입은 `Right`라고 하며, 값 자체에 더 중점을 두기 위해 `.right()` 확장 함수를 사용하는 것이 일반적입니다.

### 연산 컨텍스트 접근법
```kotlin
// 함수(fun)로 정의하므로 값을 직접 반환
fun Raise<UserNotFound>.user(): User = User(1)
```

`Raise`를 사용한 연산(함수)의 경우 값이 직접 반환됩니다.

> 💡 **Right와 Left의 의미**:
> - `Right` = 성공 (오른쪽이 "옳다"는 언어유희)
> - `Left` = 실패 (왼쪽에 남겨진 것)
>
> 다른 언어에서도 이 컨벤션을 사용합니다 (Scala, Haskell 등)

---

## 에러 발생시키기 (Raising an Error)

논리적 실패 값을 생성하려면 `Either`에서는 `left` 스마트 생성자를, `Raise` 연산 내에서는 `raise` DSL 함수를 사용합니다.
```kotlin
// 래퍼 타입 접근법
val error: Either<UserNotFound, User> = UserNotFound.left()

// 연산 컨텍스트 접근법
fun Raise<UserNotFound>.error(): User = raise(UserNotFound)
```

### ensure와 ensureNotNull

`raise`나 `left` 외에도, 불변조건을 확인하는 여러 DSL을 사용할 수 있습니다. `either { }`와 `Raise`는 Kotlin 표준 라이브러리의 `require`와 `requireNotNull`의 스타일을 반영한 **`ensure`와 `ensureNotNull`** 을 제공합니다.

예외를 던지는 대신, 술어가 충족되지 않으면 주어진 에러로 논리적 실패가 발생합니다.

#### ensure 사용하기

`ensure`는 술어와 지연 평가되는 에러 값을 받습니다. 술어가 일치하지 않으면 연산은 논리적 실패로 끝납니다.
```kotlin
data class UserNotFound(val message: String)

// 래퍼 타입 접근법
fun User.isValid(): Either<UserNotFound, Unit> = either {
    ensure(id > 0) { UserNotFound("User without a valid id: $id") }
}

// 연산 컨텍스트 접근법
fun Raise<UserNotFound>.isValid(user: User): User {
    ensure(user.id > 0) { UserNotFound("User without a valid id: ${user.id}") }
    return user
}

fun example() {
    User(-1).isValid() shouldBe UserNotFound("User without a valid id: -1").left()
    
    fold(
        { isValid(User(1)) },
        { _: UserNotFound -> fail("No logical failure occurred!") },
        { user: User -> user.id shouldBe 1 }
    )
}
```

> 💡 **`require` vs `ensure` 비교**:
> ```kotlin
> // Kotlin 표준 라이브러리 - 예외 발생
> fun validateAge(age: Int): Int {
>     require(age >= 0) { "Age must be non-negative" }
>     return age
> }
> 
> // Arrow - 타입화된 에러 반환
> fun Raise<InvalidAge>.validateAge(age: Int): Int {
>     ensure(age >= 0) { InvalidAge(age) }
>     return age
> }
> ```
>
> `require`는 `IllegalArgumentException`을 던지지만, `ensure`는 타입화된 에러를 발생시켜 컴파일 타임에 처리를 강제합니다!

#### ensureNotNull 사용하기

`ensureNotNull`은 nullable 값과 지연 평가되는 에러 값을 받습니다. 값이 null이면 논리적 실패가 발생하고, 그렇지 않으면 값이 **non-null로 스마트 캐스팅**됩니다.
```kotlin
// 래퍼 타입 접근법
fun process(user: User?): Either<UserNotFound, Long> = either {
    ensureNotNull(user) { UserNotFound("Cannot process null user") }
    user.id  // non-null로 스마트 캐스팅됨
}

// 연산 컨텍스트 접근법
fun Raise<UserNotFound>.process(user: User?): Long {
    ensureNotNull(user) { UserNotFound("Cannot process null user") }
    return user.id  // non-null로 스마트 캐스팅됨
}

fun example() {
    process(null) shouldBe UserNotFound("Cannot process null user").left()
    
    fold(
        { process(User(1)) },
        { _: UserNotFound -> fail("No logical failure occurred!") },
        { i: Long -> i shouldBe 1L }
    )
}
```

> 💡 **스마트 캐스팅의 이점**:
> `ensureNotNull` 이후에는 `!!` 연산자 없이 해당 값을 non-null로 사용할 수 있습니다. 컴파일러가 자동으로 타입을 추론합니다!

---

## 결과 실행 및 검사

### Either 값 검사하기

Kotlin의 `when`을 사용하여 `Either` 값을 검사할 수 있습니다:
```kotlin
fun example() {
    when (error) {
        is Left -> error.value shouldBe UserNotFound
        is Right -> fail("A logical failure occurred!")
    }
}
```

### fold로 모든 케이스 처리하기

`fold`는 논리적 실패와 성공 케이스 모두에 대한 람다를 제공합니다:
```kotlin
fun example() {
    fold(
        block = { error() },
        recover = { e: UserNotFound -> e shouldBe UserNotFound },
        transform = { _: User -> fail("A logical failure occurred!") }
    )
}
```

> ⚠️ **예외 처리하기**:
> 명시적으로 예외를 `Either`나 `Raise`의 일부로 잡지 않으면, 예외는 일반적인 방식으로 버블업됩니다. 예외를 처리해야 한다면, `fold`의 `catch` 인수를 사용하여 발생할 수 있는 모든 `Throwable`에서 복구할 수 있습니다.

### Raise 연산을 래퍼 타입으로 변환하기

`Raise` 연산을 래퍼 타입으로 변환하려면 **러너(runner)** 함수를 사용합니다. 각 러너는 래퍼 타입 이름의 소문자 버전입니다.
```kotlin
fun example() {
    either { error() } shouldBe UserNotFound.left()
}
```

### 래퍼 타입을 Raise 연산으로 변환하기

`Either`와 같은 타입의 값을 `Raise` 연산으로 변환하려면 **`.bind()`** 확장 함수를 사용합니다:
```kotlin
fun Raise<UserNotFound>.res(): User = user.bind()
```

### Raise DSL 사용하기 (권장 패턴)

래퍼 타입 결과를 정의할 때, 러너(`either`, `ior` 등)를 사용하고 `.bind()`로 필요한 하위 연산을 "주입"하거나 `raise`로 논리적 실패를 설명하는 것을 권장합니다.
```kotlin
val maybeTwo: Either<Problem, Int> = either { 2 }
val maybeFive: Either<Problem, Int> = either { raise(Problem) }
val maybeSeven: Either<Problem, Int> = either {
    maybeTwo.bind() + maybeFive.bind()
}
```

> 💡 **흐름 다이어그램**:
> ```
> nullable / result / either / ior
>            ↓
>       .bind()  ←→  Raise<E>.() -> A
>            ↓
>    WrapperType<E, A>
> ```

> ⚠️ **bind 호출을 잊지 마세요!**
> Arrow Detekt Rules 프로젝트에는 모든 `Either` 값에 `bind`를 호출하는지 감지하는 규칙 세트가 있습니다.

### 중첩된 에러 타입

때로는 `Either<Problem, Int?>`처럼 하나의 에러 타입이 다른 것 안에 중첩되어야 할 수 있습니다. 이 경우 **러너 함수를 타입에 나타나는 순서대로 중첩**하면 됩니다.
```kotlin
fun problematic(n: Int): Either<Problem, Int?> = either {
    nullable {
        when {
            n < 0 -> raise(Problem)   // Problem으로 폴백
            n == 0 -> raise(null)     // null로 폴백
            else -> n
        }
    }
}
```

> 💡 **raise의 출처 추적하기**:
> 프로젝트가 커지면 발생한 에러가 콜 스택을 통해 전파됩니다. 디버깅을 쉽게 하기 위해 Arrow는 `raise`와 `bind` 호출을 추적하는 방법을 제공합니다: `Raise.traced`를 참조하세요.

---

## 타입화된 에러로부터 복구하기

타입 에러를 다룰 때 두 종류의 문제를 구분하는 것이 중요합니다:

| 논리적 실패 | 예외 |
|------------|------|
| 도메인 내의 문제 | 시스템의 작동 능력에 영향을 미치는 문제 |
| 일반적인 도메인 로직으로 처리 | 예외적인 경우로만 남겨야 함 |
| 예: 존재하지 않는 사용자 찾기, 입력 데이터 유효성 검사 | 예: 데이터베이스 연결 끊김 |

역사적으로 예외는 두 경우 모두에 사용되어 왔습니다. 예를 들어, 입력 데이터가 잘못되었을 때 `UserNotValidException`을 던지는 것처럼요. **우리는 이 구분을 타입에서 명확하게 하고, 예외는 정말 예외적인 경우에만 남겨두는 것을 권장합니다.**

### 논리적 실패에서 복구하기

논리적 실패를 일으킬 수 있는 값이나 함수를 다룰 때, 폴백 값을 제공하거나 계산해야 하는 경우가 많습니다.
```kotlin
// 래퍼 타입 접근법
suspend fun fetchUser(id: Long): Either<UserNotFound, User> = either {
    ensure(id > 0) { UserNotFound("Invalid id: $id") }
    User(id)
}

// 연산 컨텍스트 접근법
suspend fun Raise<UserNotFound>.fetchUser(id: Long): User {
    ensure(id > 0) { UserNotFound("Invalid id: $id") }
    return User(id)
}
```

#### getOrElse로 복구하기

`Either` 값의 에러에서 복구하려면 `getOrElse`가 가장 편리합니다:
```kotlin
suspend fun example() {
    // 래퍼 타입 접근법
    fetchUser(-1)
        .getOrElse { e: UserNotFound -> null } shouldBe null
    
    // 연산 컨텍스트 접근법
    recover({ fetchUser(1) }) { e: UserNotFound -> null } shouldBe User(1)
}
```

> ⚠️ null로 기본값을 설정하는 것은 일반적으로 바람직하지 않습니다. 논리적 실패를 효과적으로 삼키고 에러를 무시했기 때문입니다. 그것이 바람직하다면 처음부터 nullable 타입을 사용할 수 있었을 것입니다.

#### recover로 다른 에러 타입으로 복구하기

논리적 실패를 만났을 때 적절한 폴백 값을 제공할 수 없다면, 다른 에러(`OtherError`)로 실패할 수 있는 다른 연산을 실행하고 싶을 때가 많습니다:
```kotlin
object OtherError

fun example() {
    val either: Either<OtherError, User> = fetchUser(1)
        .recover { _: UserNotFound -> raise(OtherError) }
    either shouldBe User(1).right()
    
    fetchUser(-1)
        .recover { _: UserNotFound -> raise(OtherError) } shouldBe OtherError.left()
}
```

타입 시스템은 이제 `OtherError`의 새로운 에러가 발생할 수 있음을 추적하지만, `UserNotFound`의 모든 가능한 에러에서는 복구했습니다.

> 💡 **실용적인 예시**:
> 서비스 레이어에서 데이터베이스 작업이 실패했을 때 네트워크에서 데이터를 로드하려는 경우, `DatabaseError`에서 `NetworkError`로 복구할 수 있습니다!

### 예외에서 복구하기

애플리케이션을 빌드할 때, 네트워크나 데이터베이스와 상호작용할 때처럼 부수 효과나 외부 코드를 래핑해야 하는 경우가 많습니다.

#### catch DSL 사용하기

`catch` DSL을 사용하면 외부 함수를 래핑하고 던져질 수 있는 `Throwable`을 캡처할 수 있습니다. `OutOfMemoryError`나 Kotlin의 `CancellationException`과 같은 치명적인 예외는 자동으로 캡처를 피합니다.
```kotlin
data class UserAlreadyExists(val username: String, val email: String)

// 연산 컨텍스트 접근법
suspend fun Raise<UserAlreadyExists>.insertUser(
    username: String, 
    email: String
): Long = catch({ 
    UsersQueries.insert(username, email) 
}) { e: SQLException ->
    if (e.isUniqueViolation()) raise(UserAlreadyExists(username, email))
    else throw e
}
```

#### Either.catchOrThrow 사용하기
```kotlin
// 래퍼 타입 접근법
suspend fun insertUser(
    username: String, 
    email: String
): Either<UserAlreadyExists, Long> =
    Either.catchOrThrow<SQLException, Long> { 
        UsersQueries.insert(username, email) 
    }.mapLeft { e ->
        if (e.isUniqueViolation()) UserAlreadyExists(username, email)
        else throw e
    }
```

> 💡 **이 패턴의 핵심**:
> 추적하고 싶은 예외를 타입화된 에러로 변환하고, 진정으로 예외적인 것들은 예외로 남깁니다!

---

## 에러 누적 (Accumulating Errors)

위의 모든 동작은 `Throwable`과 유사하게 작동하지만 타입화된 방식입니다. 즉, 타입화된 에러나 논리적 실패를 만나면 해당 에러가 전파되고, 연산을 계속할 수 없으며 **단락(short-circuit)** 됩니다.

컬렉션이나 `Iterable`을 다룰 때는 단락하지 않고 **모든 에러를 누적**하고 싶을 때가 많습니다.

### mapOrAccumulate 사용하기
```kotlin
data class NotEven(val i: Int)

// 연산 컨텍스트 접근법
fun Raise<NotEven>.isEven(i: Int): Int = 
    i.also { ensure(i % 2 == 0) { NotEven(i) } }

// 래퍼 타입 접근법
fun isEven2(i: Int): Either<NotEven, Int> = either { isEven(i) }
```
```kotlin
val errors = nonEmptyListOf(
    NotEven(1), NotEven(3), NotEven(5), NotEven(7), NotEven(9)
).left()

fun example() {
    // 연산 컨텍스트 접근법
    (1..10).mapOrAccumulate { isEven(it) } shouldBe errors
    
    // 래퍼 타입 접근법
    (1..10).mapOrAccumulate { isEven2(it).bind() } shouldBe errors
}
```

> 💡 **NonEmptyList (Nel)**:
> 하나 이상의 실패가 있을 수 있으므로, `Either`의 에러 타입은 일종의 리스트여야 합니다. 그러나 happy path가 아니라면 적어도 하나의 에러가 발생했음을 알고 있습니다. Arrow는 `mapOrAccumulate`의 반환 타입을 `NonEmptyList`(줄여서 `Nel`)로 만들어 이 사실을 명시적으로 표현합니다.

### 커스텀 에러 누적 로직

커스텀 타입이 있을 때 에러를 누적하는 커스텀 로직을 제공할 수도 있습니다:
```kotlin
data class MyError(val message: String)

// 두 에러 값을 하나로 결합하는 함수
operator fun MyError.plus(second: MyError): MyError = 
    MyError(message + ", ${second.message}")

// 연산 컨텍스트 접근법
fun Raise<MyError>.isEven(i: Int): Int = 
    ensureNotNull(i.takeIf { i % 2 == 0 }) { MyError("$i is not even") }

val error = MyError(
    "1 is not even, 3 is not even, 5 is not even, 7 is not even, 9 is not even"
).left()

fun example() {
    // 제공된 함수로 모든 에러를 단일 MyError 값으로 누적
    (1..10).mapOrAccumulate(MyError::plus) { isEven(it) } shouldBe error
}
```

> 💡 **forEachAccumulating**:
> 결과 값을 저장하지 않고 iterable이나 sequence의 모든 요소에 대해 에러를 발생시킬 수 있는 연산을 실행해야 한다면, `forEachAccumulating`을 사용하세요. `mapOrAccumulate`와 `forEachAccumulating`의 관계는 Kotlin 표준 라이브러리의 `map`과 `forEach`의 관계와 유사합니다.
>
> ```kotlin
> fun example() = either {
>     forEachAccumulating(1..10) { i ->
>         ensure(i % 2 == 0) { "$i is not even" }
>     }
> }
> ```

---

## 다른 연산들의 에러 누적

위의 예제에서는 요소 시퀀스에 대해 하나의 함수를 제공했습니다. 또 다른 중요하고 관련된 시나리오는 **다른 연산에서 오는 다른 에러를 누적**하는 것입니다.

예를 들어, 폼의 다른 필드에 대해 유효성 검사를 수행하고 에러를 누적해야 하지만, 각 필드에는 다른 제약 조건이 있습니다.

Arrow는 이 작업을 위해 **두 가지 스타일**을 지원합니다:
1. `zipOrAccumulate` 사용
2. `accumulate` 스코프 사용

### 예제 설정
```kotlin
data class User(val name: String, val age: Int)

// 유효성 검사에서 발생할 수 있는 문제들
sealed interface UserProblem {
    object EmptyName : UserProblem
    data class NegativeAge(val age: Int) : UserProblem
}
```

### 단락 방식 (문제점)
```kotlin
data class User private constructor(val name: String, val age: Int) {
    companion object {
        operator fun invoke(
            name: String, 
            age: Int
        ): Either<UserProblem, User> = either {
            ensure(name.isNotEmpty()) { UserProblem.EmptyName }
            ensure(age >= 0) { UserProblem.NegativeAge(age) }
            User(name, age)
        }
    }
}

fun example() {
    // 첫 번째 에러만 반환됨!
    User("", -1) shouldBe Left(UserProblem.EmptyName)
}
```

### zipOrAccumulate 사용하기
```kotlin
data class User private constructor(val name: String, val age: Int) {
    companion object {
        operator fun invoke(
            name: String, 
            age: Int
        ): Either<NonEmptyList<UserProblem>, User> = either {
            zipOrAccumulate(
                { ensure(name.isNotEmpty()) { UserProblem.EmptyName } },
                { ensure(age >= 0) { UserProblem.NegativeAge(age) } }
            ) { _, _ -> User(name, age) }
        }
    }
}
```

### accumulate 블록 사용하기
```kotlin
data class User private constructor(val name: String, val age: Int) {
    companion object {
        operator fun invoke(
            name: String, 
            age: Int
        ): Either<NonEmptyList<UserProblem>, User> = either {
            accumulate {
                ensureOrAccumulate(name.isNotEmpty()) { UserProblem.EmptyName }
                ensureOrAccumulate(age >= 0) { UserProblem.NegativeAge(age) }
                User(name, age)
            }
        }
    }
}
```

이제 문제가 올바르게 누적됩니다:
```kotlin
fun example() {
    User("", -1) shouldBe Left(
        nonEmptyListOf(UserProblem.EmptyName, UserProblem.NegativeAge(-1))
    )
}
```

> 💡 **폼 유효성 검사 UX**:
> 사용자가 폼을 제출했을 때 첫 번째 에러만 보여주는 것보다 **모든 에러를 한 번에 보여주는 것**이 훨씬 좋은 사용자 경험입니다!

> 💡 **에러 누적과 동시성**:
> 에러를 누적하는 것 외에도, `zipOrAccumulate`나 `mapOrAccumulate` 내의 각 작업을 **병렬로** 수행하고 싶을 수 있습니다. Arrow Fx는 이러한 경우를 위해 `parZipOrAccumulate`와 `parMapOrAccumulate`를 제공하며, 단락 접근 방식을 따르는 `parZip`과 `parMap`도 제공합니다.

---

## 에러 변환하기 (Transforming Errors)

모든 지점에서 시그니처가 어떤 연산에서 발생할 수 있는 에러의 타입을 명시하기 때문에 이 접근법을 "타입화된 에러"라고 부릅니다.

이 타입은 `.bind()`될 때 확인되므로, **다른 에러 타입을 가진 블록 내에서 주어진 에러 타입의 연산을 직접 소비할 수 없습니다**.

해결책은 `withError`를 사용하여 **에러를 변환**하는 것입니다:
```kotlin
val stringError: Either<String, Boolean> = "problem".left()

val intError: Either<Int, Boolean> = either {
    // String -> Int로 에러 변환
    withError({ it.length }) {
        stringError.bind()
    }
}
```

> 💡 **일반적인 패턴**:
> `withError`를 사용하여 하위 컴포넌트의 유효성 검사 에러를 더 큰 값의 유효성 검사 에러로 "연결"하는 것이 매우 일반적인 패턴입니다.

> 💡 **에러 무시하기 (ignoreErrors)**:
> `nullable`과 `Option`의 컨텍스트에서 `Either`와 같은 더 정보가 많은 타입을 소비할 때 에러 타입을 "잊어야" 하는 경우가 많습니다. `withError`로 이 동작을 달성할 수 있지만, `ignoreErrors`라는 더 선언적인 버전도 제공합니다.
