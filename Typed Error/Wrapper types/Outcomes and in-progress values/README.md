# Outcomes와 진행 중(In-progress) 값

## 개요

Arrow Core는 성공과 실패를 모델링하기 위한 세 가지 다른 타입을 포함합니다:

| 타입 | 사용 사례 |
|------|----------|
| `Option` | 실패에 대한 정보가 없을 때 |
| `Either` | 성공과 실패 케이스가 **분리**되어 있을 때 |
| `Ior` | 성공과 실패가 **동시에** 발생할 수 있을 때 |

하지만 때때로 현실은 좀 더 복잡하고, 이러한 솔루션들로는 부족할 때가 있습니다. 다행히 훌륭한 Kotlin 커뮤니티가 다른 시나리오를 다루는 라이브러리들을 개발했으며, Arrow의 `Raise` 접근 방식과 완전히 통합됩니다.

> 💡 **핵심 포인트**: 이 페이지에서 다루는 `Outcome`과 `ProgressiveOutcome`은 Arrow Core가 아닌 **커뮤니티 라이브러리**입니다. Arrow의 철학과 API 스타일을 따르면서 추가적인 사용 사례를 다룹니다.

---

## Outcomes: 부재(Absence)는 실패가 아니다

### Quiver 라이브러리

[Quiver](https://github.com/cashapp/quiver) 라이브러리는 세 가지 상태를 가진 `Outcome`을 도입합니다:

| 상태 | 의미 |
|------|------|
| `Present` | 값이 존재함 (성공) |
| `Failure` | 실패 (에러 정보 포함) |
| `Absent` | 값이 없음 (하지만 실패는 아님) |
```kotlin
val good = 3.present()           // 값이 있음: 3
val bad = "problem".failure()    // 실패: "problem" 에러
val whoKnows = Absent            // 값이 없지만 실패도 아님
```

> 💡 **Either와의 차이점**: `Either`에서는 "값이 없음"을 표현하려면 보통 `Left`(에러)로 처리해야 합니다. 하지만 **"값이 없는 것"과 "에러"는 다른 개념**입니다.

### 언제 Outcome이 필요한가?

실생활 예시를 생각해봅시다:
```kotlin
// Either로는 표현하기 어려운 케이스
fun findUserByNickname(nickname: String): ???

// 가능한 결과:
// 1. 사용자를 찾음 → Present(user)
// 2. 데이터베이스 에러 발생 → Failure(dbError)  
// 3. 해당 닉네임의 사용자가 없음 → Absent (이건 에러가 아님!)
```

`Either`를 사용하면 3번 케이스를 어떻게 표현할까요?
- `Left(UserNotFound)`? → 에러처럼 보이지만 정상적인 상황입니다
- `Right(null)`? → nullable을 피하려고 Either를 쓰는 건데...

`Outcome`을 사용하면 이 세 가지 상태를 **명확하게** 구분할 수 있습니다:
```kotlin
fun findUserByNickname(nickname: String): Outcome<DbError, User> = 
    when {
        dbError -> "DB connection failed".failure()
        userExists -> user.present()
        else -> Absent
    }
```

---

## Pedestal State로 진행 중인 값 다루기

### ProgressiveOutcome 소개

[Pedestal State](https://github.com/nicklausw/pedestal) 라이브러리는 `ProgressiveOutcome`을 도입합니다. 이 타입은 다음 두 가지를 결합합니다:
1. 계산의 **현재 상태**
2. 해당 작업이 어떻게 **진행되고 있는지**에 대한 정보

> 💡 **UI 개발에 특히 유용**: 로딩 인디케이터, 진행률 바, 스켈레톤 UI 등을 구현할 때 이 타입이 매우 유용합니다.

### 핵심 개념: 성공이 멈춤을 의미하지 않는다
```kotlin
val value = Success(5, loading(0.4))
```

이 값은 완전히 유효하며, 다음을 설명합니다:
- 작업의 마지막 성공 값은 **5**
- 새 값 검색이 **40% 완료**됨

> 💡 **실제 예시**: 소셜 미디어 피드를 생각해보세요. 이미 게시물들을 보여주고 있지만(Success), 동시에 새 게시물을 불러오는 중(Loading 40%)일 수 있습니다. 이것이 바로 `ProgressiveOutcome`이 모델링하는 상황입니다.

### ProgressiveOutcome 구조 분해

`ProgressiveOutcome`의 두 구성 요소에 접근하려면 주로 Kotlin의 **구조 분해(destructuring)**를 사용합니다.

**첫 번째 부분 (current)**: `Outcome`과 매우 유사하며 세 가지 모드가 있습니다:
- `Success` - 성공 값
- `Failure` - 실패 에러
- `Incomplete` - 아직 완료되지 않음

**두 번째 부분 (progress)**: 현재 진행 상태를 설명합니다:
- `Done` - 완료됨
- `Loading` - 로딩 중 (진행률 포함 가능)
```kotlin
fun <E, A> printProgress(po: ProgressiveOutcome<E, A>) {
    val (current, progress) = po  // 구조 분해
    
    when {
        current is Outcome.Success -> println("현재 값은 ${current.value}!")
        progress is Progress.Loading -> println("로딩 중...")
        progress is Progress.Done -> println("값을 찾지 못함 :(")
    }
}
```

### onState 헬퍼 함수

Pedestal State는 값이 해당 상태에 있을 때만 실행되는 `onState` 헬퍼 함수들을 포함합니다. 이 함수들은 **UI를 구축할 때 특히 유용**합니다. 현재 값을 보여주는 컴포넌트와 진행 상태를 설명하는 별도의 컴포넌트를 흔히 볼 수 있기 때문입니다.
```kotlin
fun <E, A> printProgress(po: ProgressiveOutcome<E, A>) {
    po.onSuccess { println("현재 값은 $it") }
    po.onLoading { println("로딩 중...") }
}
```

> 💡 **비교**: 이 패턴은 `Option`의 `onSome`/`onNone`이나 `Either`의 `onLeft`/`onRight`와 유사합니다.

### UI 컴포넌트 예시
```kotlin
// Compose UI 예시 (개념적)
@Composable
fun UserProfile(state: ProgressiveOutcome<Error, User>) {
    val (current, progress) = state
    
    Column {
        // 현재 값 표시
        current.onSuccess { user ->
            Text("이름: ${user.name}")
            Text("이메일: ${user.email}")
        }
        
        current.onFailure { error ->
            ErrorMessage(error.message)
        }
        
        current.onIncomplete {
            SkeletonLoader()
        }
        
        // 진행 상태 표시 (별도 컴포넌트)
        if (progress is Progress.Loading) {
            LinearProgressIndicator(progress = progress.percent)
        }
    }
}
```

---

## Flow와 State

### 프론트엔드 애플리케이션 패턴

프론트엔드 애플리케이션에서 유용한 패턴 중 하나는 이러한 타입들을 `Flow` 또는 `MutableState`(Compose 사용 시)와 결합하여 **시간에 따른 데이터의 진화**를 모델링하는 것입니다.
```kotlin
// Flow와 함께 사용하는 예시
class UserRepository {
    fun observeUser(id: UserId): Flow<ProgressiveOutcome<Error, User>> = flow {
        emit(Incomplete to Loading(0.0))  // 시작: 로딩 중
        
        try {
            val user = fetchUser(id)
            emit(Success(user) to Done)   // 완료: 사용자 데이터
        } catch (e: Exception) {
            emit(Failure(e.toError()) to Done)  // 실패
        }
    }
}

// ViewModel에서 사용
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _userState = MutableStateFlow<ProgressiveOutcome<Error, User>>(
        Incomplete to Loading(0.0)
    )
    val userState: StateFlow<ProgressiveOutcome<Error, User>> = _userState.asStateFlow()
    
    fun loadUser(id: UserId) {
        viewModelScope.launch {
            repo.observeUser(id).collect { _userState.value = it }
        }
    }
}
```

실제로 Pedestal State는 이 패턴을 사용하는 데 도움이 되는 함수들을 포함하는 **동반 코루틴 라이브러리**를 가지고 있습니다.

---

## 📚 추가 학습 포인트

### 타입 선택 가이드
```
값이 있거나 없다 (에러 정보 불필요)?
    → Option

성공 또는 실패 (둘 중 하나)?
    → Either

부재가 실패와 다른 의미?
    → Outcome (Quiver)

성공과 실패가 동시에 가능? (예: 경고와 함께 성공)
    → Ior

진행 상태도 함께 추적해야 함?
    → ProgressiveOutcome (Pedestal)
```

### 라이브러리 의존성
```kotlin
// Quiver (Outcome용)
implementation("app.cash.quiver:lib:x.y.z")

// Pedestal State (ProgressiveOutcome용)  
implementation("io.github.nicklausw:pedestal-state:x.y.z")
implementation("io.github.nicklausw:pedestal-state-coroutines:x.y.z")  // 코루틴 지원
```

### Arrow와의 통합

이 커뮤니티 라이브러리들은 Arrow의 `Raise` 접근 방식과 완전히 통합됩니다:
```kotlin
// Outcome을 Raise 컨텍스트에서 사용
fun example(): Outcome<Error, Int> = outcome {
    val x = someOutcome().bind()
    val y = someEither().bind()  // Either도 함께 사용 가능
    x + y
}
```