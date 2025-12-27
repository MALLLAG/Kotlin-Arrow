# 왜 Nullable 타입과 Option이 필요한가?

## 들어가며

Java를 사용해본 적이 있다면, `NullPointerException`을 한 번쯤은 만나봤을 것입니다. 보통 이런 문제는 어떤 메서드가 예상치 못하게 `null`을 반환했을 때 발생하며, 클라이언트 코드에서 그 가능성을 처리하지 않아서 생깁니다. `null` 값은 종종 "값이 없음"을 나타내기 위해 남용되곤 합니다.

Kotlin은 `null` 값을 완전히 없애고, `?` 기반의 [Null-safety 메커니즘](https://kotlinlang.org/docs/reference/null-safety.html)을 제공함으로써 이 문제를 해결합니다.

> 💡 **학습 포인트**: Kotlin의 nullable 타입(`String?`, `Int?` 등)은 컴파일 타임에 null 체크를 강제하여 NPE를 예방합니다.

## Kotlin에 이미 Nullable 타입이 있는데, 왜 Arrow의 Option이 필요할까?

`Option`을 nullable 타입 대신 사용해야 하는 경우는 **몇 가지 특수한 상황**에 한정됩니다. 그 중 하나가 바로 **중첩된 널 가능성(nested nullability)** 문제입니다.

### 문제 상황 예시

리스트의 첫 번째 요소를 반환하거나, 리스트가 비어있으면 기본값을 반환하는 `firstOrElse` 함수를 작성해봅시다:
```kotlin
fun <A> List<A>.firstOrElse(default: () -> A): A = 
    firstOrNull() ?: default()

fun example() {
    emptyList<Int?>().firstOrElse { -1 } shouldBe -1
    listOf(1, null, 3).firstOrElse { -1 } shouldBe 1
}
```

빈 리스트나 null이 아닌 값으로 시작하는 리스트에서는 예상대로 동작합니다.

### ⚠️ 예상치 못한 결과

하지만 첫 번째 값이 `null`인 리스트에서 이 함수를 실행하면 예상치 못한 결과가 나옵니다:
```kotlin
fun example() {
    listOf(null, 2, 3).firstOrElse { -1 } shouldBe null
}
```

리스트가 `isNotEmpty`이므로 첫 번째 요소인 `null`을 반환할 것으로 예상하지만, 실제로는 리스트가 `isEmpty`일 때의 기본값인 `-1`이 반환됩니다!
```
Exception in thread "main" java.lang.AssertionError: Expected null but actual was -1
```

### 왜 이런 일이 발생할까?

이것이 바로 **중첩된 널 가능성 문제(nested nullability problem)** 이며, `Option`을 사용하면 해결할 수 있습니다.

`firstOrElse` 함수 구현을 살펴보면, `firstOrNull`을 사용해서 첫 번째 요소를 가져오고, 그것이 `null`이면 기본값을 반환합니다:
```kotlin
fun <A> List<A>.firstOrElse(default: () -> A): A = 
    firstOrNull() ?: default()
```

문제는 제네릭 파라미터 `A`의 상한이 `Any?`라서 nullable 값의 리스트를 전달할 수 있다는 것입니다. 즉, 리스트의 첫 번째 요소가 `null`이면 `firstOrNull`도 `null`을 반환하는데, 이 경우를 처리하지 못합니다.

> 💡 **핵심 이해**: `firstOrNull() ?: default()`에서 `?:` 연산자는 "리스트가 비어있어서 null"인지 "첫 번째 요소가 null"인지 **구분하지 못합니다**.

### 해결 방법 1: 타입 제한 (비권장)

`A`의 상한을 `Any`로 제한할 수 있지만, 그러면 non-nullable 타입에서만 작동하게 됩니다:
```kotlin
fun <A : Any> List<A>.firstOrElse(default: () -> A): A = 
    firstOrNull() ?: default()
```

이렇게 하면 `List<Int?>` 예제가 컴파일조차 되지 않으므로 좋은 해결책이 아닙니다.

### 해결 방법 2: Option 사용 (권장)

`firstOrNone`을 사용하면 첫 번째 요소가 `null`인 경우도 처리할 수 있습니다:
```kotlin
fun <A> List<A>.firstOrElse(default: () -> A): A = 
    when(val option = firstOrNone()) {
        is Some -> option.value
        None -> default()
    }
```

이제 모든 예제가 예상대로 동작합니다. `None`을 통해 리스트가 비어있는 경우를 확실하게 감지할 수 있기 때문입니다:
```kotlin
fun example() {
    emptyList<Int?>().firstOrElse { -1 } shouldBe -1
    listOf(1, null, 3).firstOrElse { -1 } shouldBe 1
    listOf(null, 2, 3).firstOrElse { -1 } shouldBe null  // 이제 올바르게 동작!
}
```

> 💡 **Option이 필요한 다른 경우**: [RxJava](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#nulls)나 [Project Reactor](https://projectreactor.io/docs/core/release/reference/#null-safety) 같은 라이브러리는 모든 API에서 nullable 타입을 지원하지 않습니다. 이런 라이브러리와 함께 `null`을 다뤄야 한다면 `Option`을 사용하면 됩니다.

---

## Option 다루기

Arrow는 모든 타입에 대해 특별한 DSL 문법을 제공하며, nullable 타입에도 이를 제공합니다.

### Option이란?

`Option<A>`는 타입 `A`의 **선택적 값을 담는 컨테이너**입니다:
- 값이 **존재**하면: `Some<A>` 인스턴스 (현재 값을 포함)
- 값이 **없으면**: `None` 객체

> 💡 **함수형 프로그래밍 배경**: Option은 Haskell의 `Maybe`, Scala의 `Option`, Rust의 `Option<T>`과 동일한 개념입니다. "값이 있거나 없다"를 타입 시스템으로 표현합니다.

### Option 생성하기

Option을 생성하는 네 가지 방법이 있습니다:
```kotlin
// 1. 클래스 생성자 직접 사용
val some: Some<String> = Some("I am wrapped in something")
val none: None = None

// 2. 확장 함수 사용
val optionA: Option<String> = "I am wrapped in something".some()
val optionB: Option<String> = none<String>()

fun example() {
    some shouldBe optionA
    none shouldBe optionB
}
```

### Nullable 타입에서 Option 생성하기

nullable 값을 `Option`으로 변환하려면 `Option.fromNullable` 함수나 `A?.toOption()` 확장 함수를 사용합니다:
```kotlin
fun example() {
    val some: Option<String> = Option.fromNullable("Nullable string")
    val none: Option<String> = Option.fromNullable(null)
    
    "Nullable string".toOption() shouldBe some
    null.toOption<String>() shouldBe none
}
```

### ⚠️ 주의사항: 중첩된 nullable 문제

`A?`가 `null`인 경우, `Some` 또는 `.some()` 생성자를 **명시적으로** 사용해야 합니다. 그렇지 않으면 중첩된 nullable 문제로 인해 `Some` 대신 `None`을 얻게 됩니다:
```kotlin
fun example() {
    val some: Option<String?> = Some(null)  // ✅ Some(null)
    val none: Option<String?> = Option.fromNullable(null)  // ❌ None
    
    some shouldBe null.some()
    none shouldBe None
}
```

> 💡 **기억하세요**: `null` 자체를 값으로 감싸고 싶다면 반드시 `Some(null)` 또는 `null.some()`을 사용하세요!

---

## Option에서 값 추출하기

### getOrNull 사용

`Option`에서 값을 추출하는 가장 쉬운 방법은 `getOrNull`을 사용하여 nullable 타입으로 변환하는 것입니다:
```kotlin
fun example() {
    Some("Found value").getOrNull() shouldBe "Found value"
    None.getOrNull() shouldBe null
}
```

### getOrElse 사용

기본값을 제공하려면 `getOrElse`를 사용합니다. Kotlin의 `?:` 연산자와 비슷하지만, `null` 대신 `None`에 대한 기본값을 제공합니다:
```kotlin
fun example() {
    Some("Found value").getOrElse { "No value" } shouldBe "Found value"
    None.getOrElse { "No value" } shouldBe "No value"
}
```

> 💡 **비교**:
> - Kotlin nullable: `value ?: defaultValue`
> - Arrow Option: `option.getOrElse { defaultValue }`

### when을 이용한 패턴 매칭

`Option`은 `sealed class`로 모델링되어 있어서, 철저한(exhaustive) `when` 문을 사용하여 패턴 매칭할 수 있습니다:
```kotlin
fun example() {
    when(val value = 20.some()) {
        is Some -> value.value shouldBe 20
        None -> fail("$value should not be None")
    }
    
    when(val value = none<Int>()) {
        is Some -> fail("$value should not be Some")
        None -> value shouldBe None
    }
}
```

> 💡 **sealed class의 장점**: 컴파일러가 모든 케이스를 처리했는지 확인해주므로, 실수로 케이스를 빠뜨릴 수 없습니다.

---

## Option & Nullable DSL

### 문제: 중첩된 ?.let { } 블록

nullable 타입을 다룰 때, 값이 `null`인지 확인하고 처리하는 작업이 필요합니다. 보통 `?.let { }`을 사용하는데, 이는 빠르게 중첩된 `?.let { }` 블록으로 이어집니다.

사용자 도메인 클래스가 nullable 이메일 주소를 가지고 있고, id로 사용자를 찾아서 이메일을 보내고 싶다고 가정해봅시다:
```kotlin
@JvmInline value class UserId(val value: Int)
data class User(val id: UserId, val email: Email?)

fun QueryParameters.userId(): UserId? = 
    get("userId")?.toIntOrNull()?.let { UserId(it) }
fun findUserById(id: UserId): User? = TODO()
fun sendEmail(email: Email): SendResult? = TODO()
```

기존 방식:
```kotlin
fun sendEmail(params: QueryParameters): SendResult? =
    params.userId()?.let { userId ->
        findUserById(userId)?.email?.let { email ->
            sendEmail(email)
        }
    }
```

이미 상당한 중첩과 많은 `?`가 보입니다.

### 해결: bind()와 ensureNotNull 사용

Arrow의 `bind()`와 `ensureNotNull`을 사용하면 중첩을 없앨 수 있습니다:
```kotlin
fun sendEmail(params: QueryParameters): SendResult? = nullable {
    val userId = ensureNotNull(params.userId())
    val user = findUserById(userId).bind()
    val email = user.email.bind()
    sendEmail(email)
}
```

> 💡 **DSL의 마법**: `nullable { }` 블록 안에서는 마치 모든 값이 non-null인 것처럼 명령형 스타일로 코드를 작성할 수 있습니다. null이 발생하면 자동으로 블록이 종료되고 null을 반환합니다.

### nullable DSL과 Option 함께 사용하기

> 🔄 **Seamlessly mix**: `nullable` DSL은 `Option` 값에 `bind`를 호출하여 원활하게 혼합될 수 있습니다.
```kotlin
@JvmInline value class UserId(val value: Int)
data class User(val id: UserId, val email: Email?)

fun QueryParameters.userId(): UserId? = 
    get("userId")?.toIntOrNull()?.let { UserId(it) }
fun findUserById(id: UserId): Option<User> = TODO()  // Option 반환!
fun sendEmail(email: Email): SendResult? = TODO()

fun sendEmail(params: QueryParameters): SendResult? = nullable {
    val userId = ensureNotNull(params.userId())
    val user = findUserById(userId).bind()  // Option에서 bind 호출
    val email = user.email.bind()
    sendEmail(email)
}
```

### Option DSL과 nullable 타입 함께 사용하기

> 🔄 **Seamlessly mix**: `Option` DSL은 `ensureNotNull`을 사용하여 nullable 타입과 원활하게 혼합될 수 있습니다.
```kotlin
@JvmInline value class UserId(val value: Int)
data class User(val id: UserId, val email: Email?)

fun QueryParameters.userId(): Option<UserId> = 
    get("userId")?.toIntOrNull()?.let(::UserId).toOption()
fun findUserById(id: UserId): Option<User> = TODO()
fun sendEmail(email: Email): Option<SendResult> = TODO()

fun sendEmail(params: QueryParameters): Option<SendResult> = option {
    val userId = params.userId().bind()
    val user = findUserById(userId).bind()
    val email = ensureNotNull(user.email)  // nullable을 Option 컨텍스트에서 사용
    sendEmail(email).bind()
}
```

### 에러 무시하기 (ignoreErrors)

> 💡 **Ignoring errors**: `Either`처럼 더 많은 정보를 담는 타입을 사용할 때 에러 타입을 "잊어야" 할 때가 있습니다. 이를 명시적으로 표현하려면 `.bind()`를 `ignoreErrors`로 감싸세요.

---

## Option 값 검사하기

값을 추출하는 것 외에도, 그 안의 값을 **검사**만 해야 할 때가 있습니다.

### isSome과 isNone

nullable 타입에서는 `!= null`로 검사하지만, `Option`에서는 `isSome`과 `isNone`을 사용합니다:
```kotlin
fun example() {
    Some(1).isSome() shouldBe true
    none<Int>().isNone() shouldBe true
}
```

### 조건부 검사: isSome { predicate }

`Some`이 특정 조건을 만족하는 값을 포함하는지 확인할 수 있습니다. nullable 타입에서는 `?.let { } ?: false`를 사용해야 합니다:
```kotlin
fun example() {
    Some(2).isSome { it % 2 == 0 } shouldBe true   // 짝수 체크
    Some(1).isSome { it % 2 == 0 } shouldBe false
    none<Int>().isSome { it % 2 == 0 } shouldBe false
}
```

> 💡 **비교**:
> - Kotlin nullable: `value?.let { it % 2 == 0 } ?: false`
> - Arrow Option: `option.isSome { it % 2 == 0 }`
>
> Option 버전이 훨씬 읽기 쉽습니다!

### 사이드 이펙트 실행: onSome, onNone

값이 존재할 때만 사이드 이펙트를 실행하고 싶다면 `onSome`과 `onNone`을 사용합니다:
```kotlin
fun example() {
    Some(1).onSome { println("I am here: $it") }
    none<Int>().onNone { println("I am here") }
    none<Int>().onSome { println("I am not here: $it") }  // 실행 안됨
    Some(1).onNone { println("I am not here") }  // 실행 안됨
}
```

출력:
```
I am here: 1
I am here
```

> 💡 **비교**:
> - Kotlin nullable: `value?.also { println(it) }` 또는 `value?.also { if(it != null) { } }`
> - Arrow Option: `option.onSome { println(it) }`

