# Compose와 UI 개발

Arrow는 인터랙티브 애플리케이션을 개발할 때 매우 유용한 여러 기능을 제공합니다. 특히 Compose처럼 유사한 함수형 특성을 가진 라이브러리와 함께 사용할 때 시너지가 큽니다.

## 멀티플랫폼 지원

Arrow 산하의 모든 라이브러리는 **멀티플랫폼을 지원**합니다. 이는 다음과 같은 환경에서 사용할 수 있다는 의미입니다:

- **Android**: Jetpack Compose와 함께
- **Desktop, iOS, Web**: Compose Multiplatform과 함께

> 💡 **KMM 개발자를 위한 팁**: Kotlin Multiplatform Mobile(KMM) 프로젝트에서 Arrow를 사용하면 공통 모듈에서 비즈니스 로직을 함수형으로 작성하고, 각 플랫폼별 UI에서 이를 활용할 수 있습니다. 에러 처리, 데이터 변환 등의 로직을 한 번만 작성하면 됩니다!

---

## Compose는 함수형이다

상태를 가진 컴포넌트가 표준인 다른 프레임워크와 달리, **Compose가 권장하는 아키텍처는 전통적으로 함수형 접근 방식과 연관된 많은 개념을 도입**합니다.

예를 들어:
- UI는 현재 상태의 값을 인수로 받는 **함수**로 정의됩니다
- 상태 업데이트는 명시적으로 표시되며, 종종 ViewModel에 분리됩니다

결과적으로, **Arrow와 Compose는 훌륭한 파트너**가 됩니다. 아래에서는 Android 개발자(그리고 Compose Multiplatform을 통해 다른 UI 개발자)에게 즉각적인 이점이 될 몇 가지 기능을 논의합니다.

> 💡 **왜 Compose가 함수형일까?**
>
> 전통적인 Android View 시스템에서는 다음과 같이 합니다:
> ```kotlin
> // 명령형 방식 - View의 상태를 직접 변경
> textView.text = "Hello"
> button.isEnabled = true
> ```
>
> Compose에서는:
> ```kotlin
> // 선언형/함수형 방식 - 상태에 따라 UI를 "선언"
> @Composable
> fun Greeting(name: String) {
>     Text("Hello, $name")
> }
> ```
>
> UI가 상태의 함수(`UI = f(State)`)로 표현되므로, 함수형 프로그래밍의 원칙(순수 함수, 불변성 등)이 자연스럽게 적용됩니다!

---

## 더 간단한 효과적(Effectful) 코드

대부분의 애플리케이션은 독립적으로 존재하지 않습니다. 다른 서비스나 데이터 소스에 접근해야 합니다. 이런 경우 **효과적(effectful) 코드**를 작성하게 되며, `suspend`와 코루틴이 중요해집니다.

**Arrow Fx**는 여러 작업이 동시에 수행되어야 하는 코드를 단순화하는 **고수준 동시성**을 도입합니다. Structured Concurrency의 모든 규칙이 준수되도록 보장하면서도, 복잡함 없이 사용할 수 있습니다.
```kotlin
class UserSettingsModel: ViewModel() {
    private val _userData = mutableStateOf<UserData?>(null)
    val userData: State<UserData?> get() = _userData

    suspend fun loadUserData(userId: UserId) = parZip(
        { downloadAvatar(userId) },
        { UserRepository.getById(userId) }
    ) { avatarFile, user ->
        // 이 코드는 두 작업이 모두 완료된 후에 호출됩니다
        _userData.value = UserData(
            id = userId,
            details = user,
            avatar = Avatar(avatarFile)
        )
    }
}
```

> 💡 **`parZip`이란?**
>
> `parZip`은 여러 suspend 함수를 **병렬로** 실행하고, 모든 결과가 준비되면 결합합니다.
>
> 위 예제에서:
> - `downloadAvatar(userId)`와 `UserRepository.getById(userId)`가 **동시에** 실행됩니다
> - 둘 다 완료되면 마지막 람다가 호출되어 결과를 결합합니다
> - 하나라도 실패하면 나머지도 자동으로 취소됩니다 (Structured Concurrency)
>
> 만약 순차적으로 실행했다면, 총 시간은 두 작업 시간의 **합**이지만, `parZip`을 사용하면 **더 긴 쪽**의 시간만 걸립니다!

### 복원력(Resilience) 모듈

애플리케이션 외부의 모든 것은 험난한 환경입니다: 연결이 끊어지고, 서비스가 불가능해지기도 합니다. **Arrow의 resilience 모듈**은 이러한 상황을 더 잘 처리하기 위해 즉시 사용할 수 있는 여러 패턴을 제공합니다:

- **재시도 정책(Retry Policies)**: 실패 시 자동 재시도
- **서킷 브레이커(Circuit Breakers)**: 연속 실패 시 빠른 실패로 시스템 보호

> 💡 **실용 예제 - 재시도 정책**:
> ```kotlin
> // 네트워크 요청에 재시도 정책 적용
> suspend fun fetchUserWithRetry(userId: UserId): User =
>     Schedule.recurs<Throwable>(3)  // 최대 3번 재시도
>         .zipRight(Schedule.exponential(250.milliseconds))  // 지수 백오프
>         .retry { fetchUser(userId) }
> ```

---

## 내장 에러 타입

모든 애플리케이션에서 핵심적인 부분 중 하나는 **도메인 모델링**입니다. Arrow는 **불변 데이터** 사용을 강조합니다. 특히, **sealed 계층**이 다양한 상태를 설명하는 중요한 역할을 합니다.

모든 애플리케이션은 고유하지만, 인터랙티브 애플리케이션에서 흔한 시나리오는 "성공 상태"와 "에러 상태"를 갖는 것입니다. 예를 들어, 사용자 데이터를 올바르게 로드하거나, 연결 또는 인증 문제가 발생하는 경우입니다.

자체 타입을 만드는 대신, Arrow(와 형제 라이브러리인 Quiver)는 **즉시 사용 가능한 솔루션**을 제공합니다:

### Either

애플리케이션이 **완전히 성공**했거나, **어느 정도의 에러가 발생**한 모델을 설명합니다.

**유효성 검사**가 대표적인 예입니다. 데이터를 처리하기 전에 모든 필드가 유효해야 하기 때문입니다.
```kotlin
// Either<Error, Success> 형태
val result: Either<ValidationError, User> = either {
    val name = validateName(input.name).bind()
    val email = validateEmail(input.email).bind()
    User(name, email)
}
```

### Ior (Inclusive Or)

세 번째 옵션을 도입합니다: **성공했지만 그 과정에서 일부 문제가 있었던 경우**.

이 타입은 일부 잘못되거나 누락된 정보가 있어도 작업할 수 있는 도메인을 모델링하는 데 유용합니다.

> 💡 **Ior 사용 사례**:
> 예를 들어, 사용자 프로필을 로드할 때 아바타 이미지는 실패했지만 기본 정보는 성공적으로 로드된 경우, 부분적인 성공을 표현할 수 있습니다.
>
> ```kotlin
> sealed class Ior<out A, out B> {
>     data class Left<A>(val value: A) : Ior<A, Nothing>()      // 에러만
>     data class Right<B>(val value: B) : Ior<Nothing, B>()     // 성공만
>     data class Both<A, B>(val leftValue: A, val rightValue: B) : Ior<A, B>()  // 둘 다!
> }
> ```

### Outcome

**성공(Success)**, **실패(Failure)**, **부재(Absence)**를 모델링합니다.

부재 케이스는 애플리케이션이 **로딩 상태**에 있을 때 유용합니다: 아직 문제는 없지만, 데이터도 준비되지 않은 상태입니다.

> 💡 **UI 상태 관리와 Outcome**:
> ```kotlin
> sealed class Outcome<out E, out A> {
>     object Absent : Outcome<Nothing, Nothing>   // 로딩 중 또는 데이터 없음
>     data class Failure<E>(val error: E) : Outcome<E, Nothing>  // 에러 발생
>     data class Success<A>(val value: A) : Outcome<Nothing, A>  // 성공
> }
> 
> // Compose에서 사용 예:
> @Composable
> fun UserScreen(userOutcome: Outcome<Error, User>) {
>     when (userOutcome) {
>         is Outcome.Absent -> LoadingSpinner()
>         is Outcome.Failure -> ErrorMessage(userOutcome.error)
>         is Outcome.Success -> UserProfile(userOutcome.value)
>     }
> }
> ```

이러한 공통점 덕분에, Arrow는 이러한 타입들의 값을 다루기 위한 **통일된 API**를 제공합니다.

---

## 모델 업데이트하기

불변 데이터를 사용하여 상태를 모델링할 때의 잠재적인 단점 중 하나는 업데이트가 **매우 지루해질 수** 있다는 것입니다. Kotlin은 이 작업을 위해 `copy` 외에는 전용 기능을 제공하지 않기 때문입니다.

### 문제: 중첩된 copy의 지옥
```kotlin
class UserSettingsModel: ViewModel() {
    private val _userData = mutableStateOf<UserData?>(null)
    val userData: State<UserData?> get() = _userData

    fun updateName(
        newFirstName: String,
        newLastName: String
    ) {
        _userData.value = _userData.value.copy(
            details = _userData.value.details.copy(
                name = _userData.value.details.name.copy(
                    firstName = newFirstName,
                    lastName = newLastName
                )
            )
        )
    }
}
```

> 😱 **이게 뭔가요?!**
>
> 중첩된 데이터 구조를 업데이트하려면 각 레벨마다 `copy`를 호출해야 합니다. 깊이가 깊어질수록 코드는 더 복잡하고 읽기 어려워집니다. 이것을 "copy hell(복사 지옥)" 또는 "lens boilerplate"라고 부릅니다.

### 해결책: Arrow Optics

**Arrow Optics**는 불변 데이터를 조작하고 변환하는 도구를 제공하여 이러한 단점을 해결합니다. 위의 코드는 `MutableState` 전용 `copy`를 사용하여 지루한 반복 없이 다시 작성할 수 있습니다.
```kotlin
class UserSettingsModel: ViewModel() {
    private val _userData = mutableStateOf<UserData?>(null)
    val userData: State<UserData?> get() = _userData

    fun updateName(
        newFirstName: String,
        newLastName: String
    ) {
        _userData.updateCopy {
            inside(UserData.details.name) {
                Name.firstName set newFirstName
                Name.lastName set newLastName
            }
        }
    }
}
```

> 💡 **Arrow Optics의 마법**:
>
> Optics는 불변 데이터 구조의 특정 부분에 "초점을 맞추는" 방법을 제공합니다. 마치 카메라 렌즈처럼요!
>
> 핵심 개념:
> - **Lens**: 중첩된 구조의 특정 필드에 초점
> - **Prism**: sealed class의 특정 케이스에 초점
> - **Optional**: null일 수 있는 값에 초점
>
> ```kotlin
> // Optics를 사용하면 이런 "경로"를 합성할 수 있습니다
> val nameLens = UserData.details.name  // UserData → Details → Name 경로
> 
> // 그리고 이 경로를 통해 깊은 곳의 값을 쉽게 업데이트!
> ```

> 💡 **Optics 설정하기**:
>
> Arrow Optics를 사용하려면 KSP(Kotlin Symbol Processing)를 설정하고 데이터 클래스에 `@optics` 어노테이션을 추가해야 합니다:
>
> ```kotlin
> @optics
> data class UserData(
>     val id: UserId,
>     val details: Details,
>     val avatar: Avatar
> ) {
>     companion object  // Optics가 여기에 생성됨
> }
> 
> @optics
> data class Details(val name: Name, val email: String) {
>     companion object
> }
> 
> @optics
> data class Name(val firstName: String, val lastName: String) {
>     companion object
> }
> ```
