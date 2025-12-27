# Arrow 2.0 / 1.2 마이그레이션 가이드

## 먼저 1.2.0, 그 다음 2.x로

Arrow 1.x 시리즈에서 2.x 시리즈로 마이그레이션하려면, **먼저 이 가이드를 따라 버전 1.2.4로 마이그레이션하는 것을 강력히 권장**합니다. 그 후에 2.x 시리즈로 마이그레이션할 준비가 됩니다.

Arrow 1.2.0은 1.x 시리즈의 마지막 마이너 버전이며, Arrow 2.0으로의 원활한 전환까지 장기 지원 버전 역할을 합니다. **1.2.0의 모든 deprecated되지 않은 코드는 2.0.0과 소스 호환**되므로, 원할 때 천천히 그리고 우아하게 코드베이스를 Arrow 2.0.0으로 마이그레이션할 수 있습니다.

---

## Either DSL, Effect & EffectScope

Arrow 1.0.0은 `Either`와 같은 함수형 데이터 타입에서 작동하는 DSL을 도입했고, typed error를 편리하게 다루는 여러 DSL을 가능하게 했습니다. 이 DSL들은 `arrow.core.continuations` 패키지의 `Effect`와 `EffectScope` 위에 구축되었는데, 여러 문제가 있어서 Arrow 1.2.0에서 deprecated되었습니다.

### 가장 큰 문제점

**Kotlin의 `suspend` 함수와 호환되지 않았습니다.** suspend와 non-suspend 함수를 명시적으로 구분해야 했습니다.

### 해결책: 새로운 Raise DSL

Arrow 1.2.0은 새로운 **Raise DSL**을 도입하여 이 문제를 해결하고, Arrow가 typed error에 대해 전반적으로 통일된 API를 제공할 수 있게 합니다.

장점:
- API 표면이 크게 감소
- Arrow를 배우고 사용하기 더 쉬워짐
- 더 강력하고 유연한 API 구축 가능

---

## 마이그레이션 방법

기존 Either DSL에서 새로운 Raise 기반 DSL로 마이그레이션하는 방법은 두 가지가 있습니다:

1. **수동 마이그레이션** (Find + Replace 사용)
2. **반자동 마이그레이션** (KScript와 IntelliJ 사용)

---

## 수동 마이그레이션 (Find + Replace)

### Either 사용 시

#### `either { }` 교체하기
```
Find + Replace: arrow.core.continuations.either → arrow.core.raise.either
Find + Replace: arrow.core.continuations.ensureNotNull → arrow.core.raise.ensureNotNull
Find + Replace: arrow.core.computations.either → arrow.core.raise.either
Find + Replace: arrow.core.computations.ensureNotNull → arrow.core.raise.ensureNotNull
```

#### `either.eager { }` 교체하기
```
Find + Replace: arrow.core.continuations.either.eager → arrow.core.raise.either
  ⚠️ arrow.core.raise.either에 대한 중복 import가 생길 수 있음

Find + Replace: either.eager { → either {
```

#### EffectScope/EagerEffectScope 교체하기
```
Find + Replace: arrow.core.continuations.EffectScope → arrow.core.raise.Raise
Find + Replace: arrow.core.continuations.EagerEffectScope → arrow.core.raise.Raise
Find + Replace: arrow.core.continuations.ensureNotNull → arrow.core.raise.ensureNotNull
```

### Effect 사용 시
```
Find + Replace: arrow.core.continuations.Effect → arrow.core.raise.Effect
Find + Replace: arrow.core.continuations.ensureNotNull → arrow.core.raise.ensureNotNull
```

> ⚠️ `fold`, 에러 핸들러, 모든 Effect 메서드가 확장 함수로 대체되었으므로 누락된 import를 수동으로 추가해야 합니다.

### EagerEffect 사용 시
```
Find + Replace: arrow.core.continuations.EagerEffect → arrow.core.raise.EagerEffect
Find + Replace: arrow.core.continuations.ensureNotNull → arrow.core.raise.ensureNotNull
```

> ⚠️ `fold`, 에러 핸들러, 모든 EagerEffect 메서드가 확장 함수로 대체되었으므로 누락된 import를 수동으로 추가해야 합니다.

---

## 반자동 마이그레이션 (KScript와 IntelliJ)

이 마이그레이션 스크립트는 `arrow.core.computations.*`와 `arrow.core.continuations.*`를 최선의 노력으로 `arrow.core.raise.*`로 자동 마이그레이션합니다. 여러 실제 프로젝트에서 **100% 성공률**로 테스트되었으며, 전체 코드베이스를 자동으로 마이그레이션할 수 있었습니다.

### 사전 요구 사항

- `kotlinc`가 머신에 설치되어 있어야 합니다
- [공식 문서](https://kotlinlang.org/docs/command-line.html)에서 설치 방법 확인

### 주의 사항

DSL의 `ensure`와 같은 일부 메서드가 최상위 레벨이 되었고, `Effect`나 `EagerEffect`를 사용하는 경우 `fold`도 마찬가지입니다. 이러한 새로운 최상위 레벨 import는 자동으로 마이그레이션할 수 없습니다.

### 스크립트 사용법 (권장)
```bash
kotlinc -script migrate.main.kts ..
```

> ⚠️ 스크립트 실행 후 프로젝트를 컴파일하려면 **Arrow 버전 1.2.0 이상(1.2.4 권장)**이 필요합니다.

스크립트가 일부 사용하지 않는 import를 남길 수 있습니다. 이를 수정하려면:

**IntelliJ에서 Optimise Imports 실행:**
- 프로젝트 루트 또는 `src` 폴더 선택
- `⌃ ⌥ O` (Mac) 또는 `Ctrl+Alt+O` (Windows/Linux)
- 또는 프로젝트 뷰에서 우클릭 → "Optimise imports" 선택

### 대안적 방법

IntelliJ의 Optimise imports에 의존하고 싶지 않다면:

1. 마이그레이션 스크립트로 99.99%의 작업 수행
2. `./gradlew build` 실행
3. 컴파일 실패하는 파일에서 누락된 import 추가

---

## Traverse

모든 `traverse` 기능은 **Kotlin의 `map` 함수를 선호하여 deprecated**되었으며, Kotlin & IntelliJ의 `ReplaceWith`를 사용하여 자동 마이그레이션이 가능해야 합니다.

### 변경 이유

`traverse`는 FP 커뮤니티에서 매우 잘 알려진 메서드이지만, **FP 커뮤니티 외부에서는 잘 알려지지 않았습니다**. `map`을 사용하는 것이 대부분의 개발자에게 더 친숙하고, `bind`를 사용하면 나머지 DSL과 더 일관된 경험을 제공합니다.

또한, `Raise<E>` 위에서 작업할 때 `bind` 메서드가 사라지고 `map === traverse`가 됩니다.

### 예제
```kotlin
fun one(): Either<String, Int> = Either.Right(1)

// 기존 방식 (deprecated)
// val old: Either<String, List<Int>> =
//     listOf(1, 2, 3).traverse { one() }

// 새로운 방식
val new: Either<String, List<Int>> = either {
    listOf(1, 2, 3).map { one().bind() }
}
```

> 💡 **이해하기 쉽게**:
> - 기존: `traverse`라는 특별한 함수 필요
> - 신규: 익숙한 `map` + `bind` 조합
>
> 결과는 동일하지만, 코드가 더 직관적입니다!

> ⚠️ **에러 누적이 필요한 경우**: `Validated`를 사용하는 코드를 리팩토링 중이라면 아래의 "Validated & Either" 섹션을 확인하세요.

---

## Zip

`traverse`와 유사하게, 모든 `zip` 메서드도 **DSL을 선호하여 deprecated**되었습니다.

### 변경 이유

1. `zip`의 동작이 이제 `bind` 메서드로 중복됨
2. DSL이 이제 완전히 inline이므로 `zip`이 불필요
3. **arity-n 문제 해결**: `zip` 메서드는 Arrow에서 9개의 인수에 대해서만 정의되지만, 임의의 n개 인수에 대해 정의될 수 있음. DSL과 `bind`는 이 문제를 겪지 않음

> 💡 **arity-n 문제란?**
> `zip`을 10개 이상의 값에 사용하고 싶다면? Arrow에서는 `zip`이 최대 9개까지만 지원됩니다. 하지만 DSL에서는 원하는 만큼 `bind()`를 호출할 수 있습니다!

### 예제
```kotlin
fun one(): Either<String, Int> = Either.Right(1)

// 기존 방식 (deprecated)
// val old: Either<String, Int> = one().zip(one()) { x, y -> x + y }

// 새로운 방식 1: 인라인
val new: Either<String, Int> = either {
    one().bind() + one().bind()
}

// 새로운 방식 2: 변수 사용
val new2: Either<String, Int> = either {
    val x = one().bind()
    val y = one().bind()
    x + y
}
```

> ⚠️ **에러 누적이 필요한 경우**: `Validated`를 사용하는 코드를 리팩토링 중이라면 아래의 "Validated & Either" 섹션을 확인하세요.

---

## Validated & Either

Arrow 1.2.0에서 `Validated`는 `Either`를 선호하여 deprecated되었고, `ValidatedNel`은 `EitherNel`을 선호하여 deprecated되었습니다.

### 변경 이유

`Either`와 `Validated`는 타입 `E`의 에러 또는 타입 `A`의 값이라는 **동일한 추상화**를 제공합니다.

주요 차이점은 `zip`과 `traverse`가 이 데이터 타입에서 다르게 동작한다는 것입니다:
- **Validated**: `zip`과 `traverse`로 에러 누적 허용
- **Either**: 첫 번째 에러에서 단락(short-circuit)

이 동작은 새로운 Raise DSL의 구체적인 API로 연결될 수 있으며, 단일 API에서 `E`와 `NonEmptyList<E>` 모두를 지원합니다.

> 💡 **장점**: 실제로 단일 에러 `E`를 반환하는 경우 모든 반환 타입을 `NonEmptyList<E>`로 불필요하게 올릴 필요가 없습니다. 새로운 Raise DSL API 내에서 투명하게 에러를 누적할 수 있습니다.

### 반자동 마이그레이션 (ReplaceWith 사용)

**1단계**: `Validated`를 `Either`로 마이그레이션하기 전에 먼저 Raise 에러 누적 API를 활용하세요:
- `zip` → `zipOrAccumulate`
- `traverse` → `mapOrAccumulate`
- IntelliJ의 "Replace in entire project" 액션 사용

**2단계**: 나머지 API를 `Either` 동등물로 마이그레이션:
- `tapInvalid`, `withEither` 등
- `map`, `fold`, `getOrElse`와 같이 겹치는 API는 무시해도 됨

**3단계**: 모든 생성자 마이그레이션:

| Validated | Either |
|-----------|--------|
| `Validated.Valid` | `Either.Right` |
| `Validated.Invalid` | `Either.Left` |
| `A.valid()` | `A.right()` |
| `A.validNel()` | `A.right()` |
| `E.invalid()` | `E.left()` |
| `E.invalidNel()` | `E.leftNel()` |

**4단계**: `Either#toEither()` 중간 메서드를 프로젝트 전체에서 교체

### Traverse → mapOrAccumulate

`Validated`의 `traverse` 동작은 이제 `mapOrAccumulate`로 지원됩니다:
```kotlin
fun one(): Either<String, Int> = "error-1".left()
fun two(): Either<NonEmptyList<String>, Int> = 
    nonEmptyListOf("error-2", "error-3").left()

fun example() {
    // 각 요소에서 발생한 에러가 모두 누적됨
    listOf(1, 2).mapOrAccumulate { one().bind() } shouldBe 
        nonEmptyListOf("error-1", "error-1").left()
    
    listOf(1, 2).mapOrAccumulate { two().bind() } shouldBe 
        nonEmptyListOf("error-2", "error-3", "error-2", "error-3").left()
}
```

> 💡 **mapOrAccumulate vs map + bind**:
> - `map { something().bind() }`: 첫 번째 에러에서 멈춤
> - `mapOrAccumulate { something().bind() }`: 모든 에러를 수집

### Zip → zipOrAccumulate

`Validated`의 `zip` 동작은 이제 `zipOrAccumulate`로 지원됩니다:
```kotlin
fun one(): Either<String, Int> = "error-1".left()
fun two(): Either<NonEmptyList<String>, Int> = 
    nonEmptyListOf("error-2", "error-3").left()

fun example() {
    either<NonEmptyList<String>, Int> {
        zipOrAccumulate(
            { one().bind() },
            { two().bindNel() }
        ) { x, y -> x + y }
    } shouldBe nonEmptyListOf("error-1", "error-2", "error-3").left()
}
```

> 💡 **zipOrAccumulate 이해하기**:
> 폼 유효성 검사를 생각해보세요. 사용자가 폼을 제출했을 때 첫 번째 에러만 보여주는 것보다 모든 에러를 한 번에 보여주는 것이 더 좋은 UX입니다!
>
> ```kotlin
> data class UserForm(val name: String, val email: String, val age: Int)
> 
> fun validateForm(input: RawInput): Either<NonEmptyList<ValidationError>, UserForm> = 
>     either {
>         zipOrAccumulate(
>             { validateName(input.name).bind() },
>             { validateEmail(input.email).bind() },
>             { validateAge(input.age).bind() }
>         ) { name, email, age -> UserForm(name, email, age) }
>     }
> // 모든 필드의 에러가 한 번에 반환됩니다!
> ```

---

## Semigroup & Monoid

`Semigroup`과 `Monoid` 모두 Arrow 1.2.0에서 deprecated되었으며 2.0.0에서 제거될 예정입니다.

일부 deprecated된 메서드의 마이그레이션은 자동 교체 외에 추가적인 수동 단계가 필요할 수 있습니다.

### foldMap

`Iterable`, `Option`, `Either`에 대한 deprecated된 `foldMap`의 교체는 제거된 `Monoid`에 포함된 타입의 빈 값으로 `Monoid` 파라미터를 대체해야 합니다.
```kotlin
fun booleanToString(b: Boolean): String = 
    if (b) "IS TRUE! :)" else "IS FALSE.... :("

// 기존 방식 (deprecated)
fun deprecatedFoldMap() {
    val e1: Either<String, Boolean> = false.right()
    e1.foldMap(Monoid.string(), ::booleanToString) shouldBe "IS FALSE.... :("
}

// 자동 교체 실행 후 (컴파일 안됨)
fun migrateFoldMap() {
    val e1: Either<String, Boolean> = false.right()
    e1.fold({empty}, ::booleanToString) shouldBe "IS FALSE.... :("
    // empty를 찾을 수 없음!
}

// 빈 값을 추가하여 마이그레이션 완료
fun migrateFoldMap() {
    val e1: Either<String, Boolean> = false.right()
    e1.fold({""}, ::booleanToString) shouldBe "IS FALSE.... :("
}
```

> 💡 **핵심 포인트**: `Monoid.string()`이 제공하던 빈 값(`""`)을 직접 명시해야 합니다.

### combine

모든 deprecated된 `combine` 메서드는 `{a, b -> a + b}` 람다로 대체하도록 제안됩니다. 대부분의 교체에서 잘 작동하지만, 일부 경우에는 수동 수정이 필요합니다.
```kotlin
// 기존 방식 (deprecated)
fun deprecatedZip() {
    val nullableLongMonoid = object : Monoid<Long?> {
        override fun empty(): Long? = 0
        override fun Long?.combine(b: Long?): Long? = 
            nullable { this@combine.bind() + b.bind() }
    }
    val validated: Validated<Long?, Int?> = 3.valid()
    val res = validated.zip(nullableLongMonoid, Valid(Unit)) { a, _ -> a }
    res shouldBe Validated.Valid(3)
}

// 자동 교체 실행 후 (컴파일 에러)
fun migrateZip() {
    val validated: Validated<Long?, Int?> = 3.valid()
    val res = Either.zipOrAccumulate(
        { e1, e2 -> e1 + e2 },  // Long?에 대한 + 연산 없음!
        validated.toEither(),
        Valid(Unit).toEither()
    ) { a, _ -> a }.toValidated()
}

// 수동으로 nullable 처리 추가
fun migrateZip() {
    val validated: Validated<Long?, Int?> = 3.valid()
    val res = Either.zipOrAccumulate(
        { e1, e2 -> nullable { e1.bind() + e2.bind() } },
        validated.toEither(),
        Valid(Unit).toEither()
    ) { a, _ -> a }.toValidated()
    res shouldBe Validated.Valid(3)
}
```

### combineAll

`foldMap`과 유사하게, `Iterable`, `Option`, `Validate`에 대한 deprecated된 `combineAll`의 교체는 `fold` 메서드에서 initial 파라미터를 수동으로 추가해야 합니다.
```kotlin
// 기존 방식 (deprecated)
fun deprecatedCombineAll() {
    val l: List<Int> = listOf(1, 2, 3, 4, 5)
    l.combineAll(Monoid.int()) shouldBe 10
}

// 자동 교체 실행 후 (컴파일 안됨)
fun migrateCombineAll() {
    val l: List<Int> = listOf(1, 2, 3, 4, 5)
    l.fold(initial) { a1, a2 -> a1 + a2 } shouldBe 10
    // initial을 찾을 수 없음!
}

// initial 값을 추가하여 마이그레이션 완료
fun migrateCombineAll() {
    val l: List<Int> = listOf(1, 2, 3, 4, 5)
    l.fold(0) { a1, a2 -> a1 + a2 } shouldBe 10
}
```

### replicate

`Option`과 `Either`에 대한 deprecated된 `Monoid`를 제거할 때 `replicate`도 약간의 도움이 필요합니다.
```kotlin
// 기존 방식 (deprecated)
fun deprecatedReplicate() {
    val rEither: Either<String, Int> = 125.right()
    val n = 3
    rEither.replicate(n, Monoid.int()) shouldBe Either.Right(375)
}

// 자동 교체 실행 후 (컴파일 안됨)
fun migrateReplicate() {
    val rEither: Either<String, Int> = 125.right()
    val n = 3
    val res = if (n <= 0) Either.Right(initial)
    else rEither.map { b -> 
        List<Int>(n) { b }.fold(initial) { r, t -> r + t } 
    }
    // initial을 찾을 수 없음!
    res shouldBe Either.Right(375)
}

// 빈 값을 추가하여 마이그레이션 완료
fun migrateReplicate() {
    val rEither: Either<String, Int> = 125.right()
    val n = 3
    val res = if (n <= 0) Either.Right(0)
    else rEither.map { b -> 
        List<Int>(n) { b }.fold(0) { r, t -> r + t } 
    }
    res shouldBe Either.Right(375)
}
```

---

## Ior

대부분의 `Ior` 데이터 타입의 deprecated된 메서드 마이그레이션, 특히 `traverse`와 `crosswalk` 관련은 **수동으로 교체**해야 합니다.

주된 이유는 제네릭을 사용할 때 IntelliJ가 일부 타입을 추론하는 방법을 모르기 때문입니다. 이 상황이 약간 성가실 수 있지만, Arrow 소스 코드를 탐색하고 더 많은 전문 지식을 얻을 수 있는 좋은 기회입니다.

### crosswalk

`Ior`의 `crosswalk` 구현:
```kotlin
public inline fun <A, B, C> Ior<A, B>.crosswalk(
    fa: (B) -> Iterable<C>
): List<Ior<A, C>> = fold(
    { emptyList() },
    { b -> fa(b).map { Ior.Right(it) } },
    { a, b -> fa(b).map { Ior.Both(a, it) } }
)
```

마이그레이션 예제:
```kotlin
// 기존 방식 (deprecated)
fun deprecatedCrosswalk() {
    val rightIor: Ior<String, Int> = Ior.Right(124)
    val result = rightIor.crosswalk { listOf(it) }
    result shouldBe listOf(Ior.Right(124))
}

// fold 구현을 사용하여 수동 교체
fun migrateCrosswalk() {
    val rightIor: Ior<String, Int> = Ior.Right(124)
    val result = rightIor.fold(
        { emptyList<Int>() },
        { b -> listOf(b).map { Ior.Right(it) } },
        { a, b -> listOf(b).map { Ior.Both(a, it) } }
    )
    result shouldBe listOf(Ior.Right(124))
}
```

### traverse

`Option`을 반환하는 함수에 대한 `Ior` `traverse` 메서드도 유사한 상황입니다.

`Ior`의 `traverse` 구현:
```kotlin
public inline fun <A, B, C> Ior<A, B>.traverse(
    fa: (B) -> Option<C>
): Option<Ior<A, C>> {
    return fold(
        { a -> Some(Ior.Left(a)) },
        { b -> fa(b).map { Ior.Right(it) } },
        { a, b -> fa(b).map { Ior.Both(a, it) } }
    )
}
```

마이그레이션 예제:
```kotlin
fun evenOpt(i: Int): Option<Int> = 
    if (i % 2 == 0) i.some() else None

// 기존 방식 (deprecated)
fun deprecatedTraverse() {
    val rightIor: Ior<String, Int> = Ior.Right(124)
    val result = rightIor.traverse { evenOpt(it) }
    result shouldBe Some(Ior.Right(124))
}

// fold 구현을 사용하여 수동 교체
fun migrateTraverse() {
    val rightIor: Ior<String, Int> = Ior.Right(124)
    val result = rightIor.fold(
        { a -> Some(Ior.Left(a)) },
        { b -> evenOpt(b).map { Ior.Right(it) } },
        { a, b -> evenOpt(b).map { Ior.Both(a, it) } }
    )
    result shouldBe Some(Ior.Right(124))
}
```
