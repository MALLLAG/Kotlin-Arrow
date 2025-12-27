# Either & Ior (& Result)

## 개요

`Either<E, A>`와 `Ior<E, A>` (Inclusive Or)는 모두 타입 `E` 또는 `A`의 값을 담을 수 있습니다. 관례적으로:
- 타입 `E`는 **에러**를 나타냅니다
- 타입 `A`는 **성공**을 나타냅니다

예를 들어, `Either<DbError, User>`는 데이터베이스에 접근하여 `User`를 반환하지만 `DbError`로 실패할 수도 있는 함수의 좋은 결과 타입이 될 수 있습니다.

> 💡 **함수형 프로그래밍 배경**: Either는 Haskell, Scala 등에서 온 개념으로, 예외(Exception) 대신 타입 시스템을 통해 에러를 명시적으로 표현하는 방법입니다.

### Either vs Ior 비교

| 타입 | 가능한 상태 | 사용 사례 |
|------|------------|----------|
| `Either<E, A>` | `Left(E)` 또는 `Right(A)` | 실패 또는 성공 (둘 중 하나) |
| `Ior<E, A>` | `Left(E)`, `Right(A)`, 또는 `Both(E, A)` | 경고와 함께 성공하는 경우 포함 |

**Either**는 두 가지 가능성만 허용합니다:
- `Left`: 타입 `E`의 값 (에러)
- `Right`: 타입 `A`의 값 (성공)

**Ior**는 세 번째 옵션인 `Both`를 제공합니다. `Both`를 사용하면 성공으로 간주되지만 실행 중 일부 잠재적 에러가 있는 상태를 표현할 수 있습니다. 예를 들어, 성공적으로 완료되었지만 경고가 있는 컴파일러 같은 경우입니다.

> 💡 **실무 팁**: `Ior`는 자주 사용되지 않습니다. 대부분의 경우 `Either`로 충분합니다. `Ior`는 "부분적 성공" 개념이 필요할 때만 고려하세요.

### 표준 라이브러리의 Result와의 비교

이 두 타입은 표준 라이브러리의 `Result`와 매우 비슷해 보입니다. 그러나 `Result`는 언어의 코루틴과 예외 메커니즘에 **밀접하게 결합**되어 있습니다. 사실, "실패" 케이스는 `Throwable` 에러만 허용합니다.

| 특징 | Either | Result |
|------|--------|--------|
| 에러 타입 | 모든 타입 가능 (`E`) | `Throwable`만 가능 |
| 유연성 | 높음 | 제한적 |
| 코루틴 통합 | Arrow 통합 필요 | 기본 제공 |

> 💡 **언제 무엇을 사용할까?**
> - 간단한 예외 처리: `Result`
> - 도메인 특화 에러 타입이 필요한 경우: `Either`
> - 경고와 함께 성공이 필요한 경우: `Ior`

---

## 빌더 사용하기

`Either`와 `Ior`를 다루는 **권장 방법**은 빌더를 사용하는 것입니다. `either` 또는 `ior` 호출과 람다로 시작합니다. 해당 블록 내에서 `raise`, `ensure`, `recover` 같은 통합된 typed errors API에 접근할 수 있습니다.

`Result` 타입의 경우 빌더는 `result`라고 불리지만, 대부분의 에러는 `runCatching`에서 옵니다.
```kotlin
import arrow.core.raise.either
import arrow.core.raise.ensure

data class MyError(val message: String)

fun isPositive(i: Int): Either<MyError, Int> = either {
    ensure(i > 0) { MyError("$i is not positive") }
    i
}

suspend fun example() {
    isPositive(-1) shouldBe MyError("-1 is not positive").left()
    isPositive(1) shouldBe 1.right()
}
```

> 💡 **ensure 이해하기**: `ensure(조건) { 에러 }` 는 조건이 `false`면 에러를 발생시키고, `true`면 계속 진행합니다. Java의 `if (!condition) throw error` 패턴과 비슷하지만, 예외 대신 `Either`를 반환합니다.

### Raise 컨텍스트 이해하기

전체적인 그림을 보면, 이러한 블록 내부에서 잠재적 에러는 `Raise<E>` 타입의 리시버로 표현됩니다. 해당 리시버가 있는 함수는 다양한 타입으로 변환될 수 있습니다:
- `Either`
- `Ior`
- `Result`
- `Option`
- nullable 타입
```
either / ior / result
        ↓
   .bind() 사용
        ↓
Raise<E>.() -> A  →  Either<E, A> / Ior<E, A>
```

### bind()로 값 언래핑하기

일반적인 시나리오는 잠재적으로 에러가 있는 값(블록 내부의 것만이 아니라)을 **언래핑**하고 싶은 경우입니다. 즉, 그 값의 잠재적 에러가 전체 블록의 에러로 **버블업**되거나, 값이 성공을 나타내면 실행을 계속하고 싶습니다. 이런 경우 해당 값에 `.bind()`를 호출해야 합니다.

> 💡 **실무 팁**: Detekt용 커스텀 `NoEffectScopeBindableValueAsStatement` 규칙을 사용하면 `either` 또는 `ior` 블록 내에서 `.bind()` 호출을 잊는 것을 방지할 수 있습니다.

> ⚠️ **자주 하는 실수**: `either` 블록 안에서 `Either` 타입의 값을 사용할 때 `.bind()` 호출을 잊으면, 에러가 전파되지 않고 `Either` 객체 자체가 값으로 사용됩니다!
```kotlin
// ❌ 잘못된 예
either {
    val result = someFunction() // Either<Error, Value>를 반환
    result // Either 객체 자체가 반환됨!
}

// ✅ 올바른 예
either {
    val result = someFunction().bind() // Value가 추출됨
    result
}
```

---

## Ior 에러 결합하기

`Either` 블록의 흐름은 간단합니다:
1. 각 코드 라인을 실행합니다
2. 어느 시점에서 `Left`를 `bind()`하거나 `raise`를 만나면, 멈추고 그 값을 반환합니다
3. 끝까지 도달하면, 결과를 `Right`로 감쌉니다

`ior` 블록은 좀 더 복잡합니다. 보고할 에러가 있지만 실행을 계속할 값도 있는 상황이 생길 수 있기 때문입니다. 이는 질문을 제기합니다: **블록의 여러 단계가 Both인 경우 어떻게 해야 할까요?**

현재 API는 그 답을 개발자에게 맡깁니다. `ior` 빌더에는 **여러 에러를 어떻게 결합할지** 지정하는 추가 파라미터가 있습니다.

> 💡 **Ior 에러 결합 예시**:
> ```kotlin
> ior(combineError = { e1, e2 -> e1 + e2 }) {
>     // 여러 Both가 발생하면 에러들이 결합됨
> }
> ```

---

## 빌더 없이 사용하기

일부 시나리오에서는 빌더가 과도할 수 있습니다. 그런 경우를 위해 `Either`와 `Ior`를 직접 생성하거나 조작하는 함수들을 제공합니다.

### 생성: .left()와 .right() 확장 함수

`.left()`와 `.right()` 같은 확장 함수는 생성자만큼 내부 내용을 가리지 않는 표현식을 작성하는 또 다른 방법을 제공합니다. 유효성 검사는 종종 이 스타일로 작성됩니다.
```kotlin
// 구성하려는 타입
@JvmInline value class Age(val age: Int)

// 잠재적 문제들
sealed interface AgeProblem {
    object InvalidAge: AgeProblem
    object NotLegalAdult: AgeProblem
}

// 유효성 검사는 문제 또는 구성된 값을 반환
fun validAdult(age: Int): Either<AgeProblem, Age> = when {
    age < 0 -> AgeProblem.InvalidAge.left()
    age < 18 -> AgeProblem.NotLegalAdult.left()
    else -> Age(age).right()
}
```

> 💡 **패턴 이해**: 이 패턴은 "Smart Constructor" 또는 "Validation" 패턴이라고 불립니다. 유효하지 않은 상태의 객체가 생성되는 것을 타입 시스템 수준에서 방지합니다.

### Either.catch로 예외 감싸기

`Either`를 얻는 또 다른 방법은 `Either.catch`를 사용하는 것입니다. 예외를 던질 수 있는 계산을 감싸고, 그런 경우 `Left`를 반환합니다. 본질적으로 표준 라이브러리의 `runCatching`과 같지만, `Result` 대신 `Either`를 반환합니다.
```kotlin
// 예외를 던질 수 있는 코드를 Either로 감싸기
val result: Either<Throwable, Int> = Either.catch {
    "123".toInt()
}
```

### 기타 유용한 함수들

나머지 API는 typed errors의 것과 밀접하게 따릅니다. 예를 들어, 추가적인 `either { }` 블록 없이 `Either`에서 직접 `recover`나 `zipOrAccumulate`를 호출할 수 있습니다.

#### mapLeft: 에러 변환하기

빌더의 일부가 아닌 잠재적으로 유용한 함수는 `mapLeft`입니다. 값이 에러를 나타낼 때 함수를 적용합니다. 이 시나리오는 코드에 다른 에러 타입의 계층 구조가 있을 때 자주 발생합니다.
```kotlin
// 에러 타입 변환 예시
fun fetchUser(): Either<DbError, User> = TODO()

fun getUserName(): Either<AppError, String> =
    fetchUser()
        .mapLeft { dbError -> AppError.DatabaseFailed(dbError) }  // 에러 타입 변환
        .map { user -> user.name }  // 성공 값 변환
```

> 💡 **map vs mapLeft**:
> - `map`: `Right` 값(성공)을 변환
> - `mapLeft`: `Left` 값(에러)을 변환

---

## 유효성 검사를 위한 Either

`Either`는 **두 가지 얼굴**을 가집니다:
1. 예외처럼 코드의 문제를 모델링하는 데 사용
2. 입력 데이터에 대한 **유효성 검사**를 정의하는 데 사용

이 두 시나리오의 차이는 코드에서 여러 문제가 발생할 때 **어떻게 반응하느냐**입니다.

### 1. Fail-Fast (빠른 실패) 접근법

예외를 생각할 때, 우리는 **fail-fast** 또는 **fail-first** 접근법을 가집니다:
- 문제를 발견하면 **즉시** 실행을 중단하고 호출자에게 보고합니다
- 이런 시나리오에서 단계들은 서로 **의존**하므로 계속 시도하는 것이 의미 없습니다

> 💡 **예시**: 사용자 조회 → 권한 확인 → 데이터 업데이트  
> 사용자 조회가 실패하면 이후 단계는 의미가 없습니다.

### 2. Accumulation (축적) 접근법

유효성 검사를 생각할 때, 입력 데이터의 잠재적 문제에 대해 **가능한 한 포괄적**이고 싶습니다:
- 주어진 이름과 나이가 둘 다 잘못되었다면, **첫 번째 것만 아니라 둘 다** 보고하고 싶습니다
- 이 접근법은 **축적(accumulation)** 이라고 불리며, 실패가 서로 **독립적인** 계산일 때 발생합니다

> 💡 **예시**: 회원가입 폼 유효성 검사  
> 이름이 비어있고, 이메일이 잘못되었고, 비밀번호가 너무 짧다면 → 세 가지 에러를 모두 한 번에 보여주는 것이 좋은 UX입니다.

### 기본 동작과 에러 축적

기본적으로 `either` 블록은 **첫 번째 접근법**(fail-fast)을 따릅니다.

에러를 **축적**하고 싶다면 `zipOrAccumulate` 또는 `mapOrAccumulate`를 사용해야 합니다:

| 함수 | 사용 사례 |
|------|----------|
| `zipOrAccumulate` | 다른 계산들을 인자로 받음, 각각 다른 타입 반환 가능 |
| `mapOrAccumulate` | `Iterable`의 요소들에 동일한 계산을 균일하게 적용 |
```kotlin
// zipOrAccumulate 예시
fun validateUser(name: String, email: String, age: Int): Either<Nel<ValidationError>, User> =
    zipOrAccumulate(
        { validateName(name).bind() },
        { validateEmail(email).bind() },
        { validateAge(age).bind() }
    ) { validName, validEmail, validAge ->
        User(validName, validEmail, validAge)
    }
```

### EitherNel: 비어있지 않은 에러 리스트

유효성 검사를 설명할 때 흔한 패턴은 에러 타입으로 `List<Problem>`을 가진 `Either`를 갖는 것입니다. Arrow는 `Left` 값이 있지만 문제 목록이 **비어있는** 어색한 상황이 절대 발생하지 않도록 보장하는 더 세련된 버전을 제공합니다.
```kotlin
public typealias EitherNel<E, A> = Either<NonEmptyList<E>, A>
```

> 💡 **Nel = NonEmptyList**: 최소 하나의 요소가 보장되는 리스트입니다. 유효성 검사에서 에러가 있다면 최소 하나는 있어야 하므로, 이 타입이 더 정확합니다.

### Arrow 1.x vs 2.x

Arrow 1.x 시리즈에서는 `Validated`라는 다른 타입이 에러 축적 전략을 구현했습니다. 그러나 API가 거의 동일했고, 때때로 코드가 `Either`와 `Validated` 사이의 변환으로 넘쳐났습니다.

Arrow 2.x는 대신 단일 `Either` 타입을 제공하지만, 유효성 검사를 설명하는 경우 `EitherNel` 타입 별칭을 사용하는 것을 권장합니다.

---

## 📚 추가 학습 포인트

### Either 사용 시 자주 하는 실수들

1. **bind() 호출 잊기**
```kotlin
// ❌ 잘못된 예
either {
    val user = findUser(id)  // Either<Error, User> 반환
    user.name  // 컴파일 에러 또는 예상치 못한 동작
}

// ✅ 올바른 예
either {
    val user = findUser(id).bind()
    user.name
}
```

2. **에러 타입 불일치**
```kotlin
// ❌ 잘못된 예 - 다른 에러 타입
fun getUser(): Either<UserError, User> = TODO()
fun getOrder(): Either<OrderError, Order> = TODO()

either {
    val user = getUser().bind()  // UserError
    val order = getOrder().bind()  // OrderError - 타입 불일치!
}

// ✅ 올바른 예 - 공통 에러 타입 사용
sealed interface AppError
data class UserError(...): AppError
data class OrderError(...): AppError
```

### 실무에서의 Either 패턴
```kotlin
// 서비스 레이어 예시
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    fun registerUser(request: RegisterRequest): Either<RegistrationError, User> = either {
        // 1. 유효성 검사
        val validatedEmail = validateEmail(request.email).bind()
        val validatedPassword = validatePassword(request.password).bind()
        
        // 2. 중복 체크
        ensure(!userRepository.existsByEmail(validatedEmail)) {
            RegistrationError.EmailAlreadyExists
        }
        
        // 3. 사용자 생성
        val user = userRepository.save(User(validatedEmail, validatedPassword))
        
        // 4. 환영 이메일 발송 (실패해도 사용자 등록은 성공)
        emailService.sendWelcome(user.email)
            .onLeft { logger.warn("Failed to send welcome email: $it") }
        
        user
    }
}
```
