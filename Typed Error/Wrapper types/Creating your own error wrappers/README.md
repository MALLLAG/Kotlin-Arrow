# 나만의 에러 래퍼 만들기

## 개요

`Raise`는 타입화된 에러를 발생시키는 **나만의 DSL을 만들 수 있게** 해주는 강력한 도구입니다. `Either`와 같은 유사한 데이터 타입을 제공하는 기존 라이브러리 및 프레임워크, 또는 **자신만의 커스텀 타입**과 쉽게 통합할 수 있습니다.

---

## 예제: LCE 타입

프론트엔드에서 자주 사용되는 인기 있는 ADT(Algebraic Data Type)를 예로 들어봅시다. **Loading**, **Content**, **Failure**를 모델링하는 타입으로, 흔히 **LCE**로 약칭됩니다.
```kotlin
sealed interface Lce<out E, out A> {
    object Loading : Lce<Nothing, Nothing>
    data class Content<A>(val value: A) : Lce<Nothing, A>
    data class Failure<E>(val error: E) : Lce<E, Nothing>
}
```

> 💡 **LCE 패턴 이해하기**: 이 패턴은 UI 상태 관리에서 매우 흔합니다:
> - `Loading`: 데이터를 불러오는 중
> - `Content`: 데이터 로딩 성공
> - `Failure`: 에러 발생

### 다른 UI 상태 타입들과의 비교
```kotlin
// 흔히 볼 수 있는 비슷한 패턴들
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

// Resource 패턴 (Android에서 자주 사용)
sealed class Resource<out T> {
    object Loading : Resource<Nothing>()
    data class Success<T>(val data: T) : Resource<T>()
    data class Failure(val throwable: Throwable) : Resource<Nothing>()
}
```

---

## 기본 기능 구현

`Failure` 또는 `Loading` 케이스를 만나면 **단락(short-circuit)** 하고 계산을 계속하지 않도록 하고 싶다고 가정해봅시다. 이를 수행하는 `Lce`용 `Raise` 인스턴스를 정의하는 것은 쉽습니다.

### LceRaise 클래스 만들기

context receiver 없이 **컴포지션 패턴**을 사용합니다. `Lce.Loading`과 `Lce.Failure` 모두를 raise해야 하므로, `Raise` 인스턴스는 `Lce<E, Nothing>`을 raise할 수 있어야 합니다. 이것을 `LceRaise` 클래스로 감쌉니다.

이 클래스 내에서 `bind` 함수를 정의하여 만나는 `Failure` 또는 `Loading` 케이스를 단락시키거나, 그렇지 않으면 `Content` 값을 반환합니다.
```kotlin
@JvmInline
value class LceRaise<E>(val raise: Raise<Lce<E, Nothing>>) : Raise<Lce<E, Nothing>> by raise {
    
    fun <A> Lce<E, A>.bind(): A = when (this) {
        is Lce.Content -> value
        is Lce.Failure -> raise.raise(this)
        Lce.Loading -> raise.raise(Lce.Loading)
    }
}
```

> 💡 **코드 분석**:
> - `@JvmInline value class`: 런타임 오버헤드 없이 래퍼 제공
> - `Raise<Lce<E, Nothing>> by raise`: 위임을 통해 `Raise` 인터페이스 구현
> - `bind()`: `Lce`에서 값을 추출하거나 에러/로딩 상태면 단락

### DSL 함수 만들기

이제 DSL 함수만 있으면 됩니다. `recover` 또는 `fold` 함수를 사용하여 `Raise` 타입 클래스에서 `RaiseLce<E, Nothing>` 인스턴스를 얻을 수 있습니다.

블록을 `Lce.Content` 값으로 감싸고, 만나는 `Lce<E, Nothing>` 값을 반환합니다. `Raise<Lce<E, Nothing>>`를 `LceRaise`로 감싸서 블록을 호출할 수 있습니다.
```kotlin
@OptIn(ExperimentalTypeInference::class)
inline fun <E, A> lce(@BuilderInference block: LceRaise<E>.() -> A): Lce<E, A> =
    recover({ Lce.Content(block(LceRaise(this))) }) { e: Lce<E, Nothing> -> e }
```

> 💡 **작동 원리**:
> 1. `recover` 블록 내에서 `LceRaise`를 생성하여 `block`에 전달
> 2. 성공하면 결과를 `Lce.Content`로 감쌈
> 3. `Failure`나 `Loading`이 raise되면 그 값을 그대로 반환

---

## DSL 사용하기

이제 이 DSL을 사용하여 계산과 `Lce` 값을 위에서 논의한 것과 같은 방식으로 조합할 수 있습니다. 또한, 이 DSL은 `Raise` 위에 구축되었으므로 위에서 논의한 모든 함수(`ensure`, `bind` 등)를 사용할 수 있습니다.
```kotlin
fun example() {
    // 성공 케이스: 두 Content 값을 바인딩하고 더하기
    lce {
        val a = Lce.Content(1).bind()
        val b = Lce.Content(1).bind()
        a + b
    } shouldBe Lce.Content(2)
    
    // 실패 케이스: ensure로 조건 검사
    lce {
        val a = Lce.Content(1).bind()
        ensure(a > 1) { Lce.Failure("a is not greater than 1") }
        a + 1
    } shouldBe Lce.Failure("a is not greater than 1")
}
```

> 💡 **핵심 포인트**: `either { }` 블록에서 사용하던 것과 **동일한 패턴**을 `lce { }` 블록에서도 사용할 수 있습니다!

### 실제 사용 예시
```kotlin
// ViewModel에서 사용하는 예시
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _state = MutableStateFlow<Lce<String, User>>(Lce.Loading)
    val state: StateFlow<Lce<String, User>> = _state.asStateFlow()
    
    fun loadUser(id: UserId) {
        viewModelScope.launch {
            _state.value = lce {
                // API 호출 결과를 Lce로 변환
                val response = repo.fetchUser(id)  // Lce<String, User> 반환
                val user = response.bind()         // Content면 값 추출, 아니면 단락
                
                // 추가 검증
                ensure(user.isActive) { Lce.Failure("User is not active") }
                
                user
            }
        }
    }
}
```

---

## Context Parameters를 사용한 더 간단한 버전

> 💡 **참고**: Context parameters는 Kotlin의 실험적 기능입니다. 현재는 context receivers로 알려져 있으며, 향후 버전에서 정식 지원될 예정입니다.

context parameters를 사용했다면, 이 DSL을 정의하는 것이 더 간단해지고 `Raise` 타입 클래스를 직접 사용할 수 있습니다:
```kotlin
context(_: Raise<Lce<E, Nothing>>)
fun <E, A> Lce<E, A>.bind(): A = when (this) {
    is Lce.Content -> value
    is Lce.Failure -> raise(this)
    Lce.Loading -> raise(Lce.Loading)
}

inline fun <E, A> lce(@BuilderInference block: Raise<Lce<E, Nothing>>.() -> A): Lce<E, A> =
    recover({ Lce.Content(block(this)) }) { e: Lce<E, Nothing> -> e }
```

---

## Failure에 대한 고찰

### 왜 Lce<E, Nothing>을 선택했는가?

`Lce<E, Nothing>`을 Failure 타입으로 선택한 이유는 **여러 에러를 가진 DSL**을 허용하기 때문입니다.

### DialogResult 예제

`Lce`와 비슷하지만 성공으로 간주되지 않는 추가 상태가 있는 타입을 고려해봅시다:
```kotlin
// 평면적(flat) 구조
DialogResult<out T>
├── Positive<out T>(value: T) : DialogResult<T>  // 확인
├── Neutral : DialogResult<Nothing>               // 나중에
├── Negative : DialogResult<Nothing>              // 아니오
└── Cancelled: DialogResult<Nothing>              // 취소
```

이 **평면적인 타입** `DialogResult`에 대해 `Raise`를 편리하게 제공할 수 없고, 어쩔 수 없이 `DialogResult<Nothing>`을 사용해야 합니다.

### 더 나은 타입 계층 구조

하지만 타입을 **다르게 계층화**하면:
```kotlin
// 계층적 구조
DialogResult<out T>
├── Positive<out T>(value: T) : DialogResult<T>
└── Error : DialogResult<Nothing>
    ├── Neutral : Error
    ├── Negative : Error
    └── Cancelled: Error
```

이제 `Raise<DialogResult.Error>`의 이점을 다시 얻을 수 있습니다!

> 💡 **핵심 통찰**: 에러 타입들을 공통 부모 아래에 그룹화하면 `Raise` 시스템과 더 잘 통합됩니다.

### Either와의 상호운용

이렇게 설계하면 **Either와도 상호운용**할 수 있습니다!
```kotlin
dialogResult {
    val x: Int = DialogResult.Positive(1).bind()
    val y: Int = DialogResult.Error.left().bind()  // Either와 함께 사용!
    x + y
}
```

### 에러 축적(Accumulation)과 함께 사용

에러를 축적하고 싶다면, Kotlin의 기본 동작을 활용할 수 있습니다:
```kotlin
fun dialog(int: Int): DialogResult<Int> =
    if (int % 2 == 0) DialogResult.Positive(int) 
    else DialogResult.Neutral

val res: Either<NonEmptyList<DialogResult.Error>, NonEmptyList<Int>> =
    listOf(1, 2, 3).mapOrAccumulate { i: Int ->
        dialog(i).getOrElse { raise(it) }
    }

dialogResult {
    res.mapLeft { errors -> 
        // 축적된 에러들 처리
        errors.joinToString { it.toString() }
    }.bind()
}
```

---

## 나만의 래퍼 만들기 체크리스트

커스텀 에러 래퍼를 만들 때 따라야 할 단계:

### 1단계: sealed 타입 정의
```kotlin
sealed interface MyResult<out E, out A> {
    data class Success<A>(val value: A) : MyResult<Nothing, A>
    data class Failure<E>(val error: E) : MyResult<E, Nothing>
    // 필요한 다른 상태들...
}
```

### 2단계: Raise 래퍼 클래스 생성
```kotlin
@JvmInline
value class MyResultRaise<E>(val raise: Raise<MyResult<E, Nothing>>) 
    : Raise<MyResult<E, Nothing>> by raise {
    
    fun <A> MyResult<E, A>.bind(): A = when (this) {
        is MyResult.Success -> value
        is MyResult.Failure -> raise.raise(this)
    }
}
```

### 3단계: DSL 빌더 함수 생성
```kotlin
inline fun <E, A> myResult(block: MyResultRaise<E>.() -> A): MyResult<E, A> =
    recover({ MyResult.Success(block(MyResultRaise(this))) }) { e -> e }
```

### 4단계: 사용!
```kotlin
val result = myResult {
    val x = MyResult.Success(1).bind()
    val y = MyResult.Success(2).bind()
    x + y
}
```
