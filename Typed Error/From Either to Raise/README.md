# Either에서 Raise로

## 개요

다른 함수형 프로그래밍 생태계에서 타입이 있는 에러(Typed Errors)는 주로 `Either`와 같은 전용 타입을 중심으로 다루어집니다. 이 타입들은 계산이 성공하거나 에러로 끝날 수 있음을 표현할 수 있게 해줍니다. Arrow에서도 이 스타일을 완전히 지원하지만, **`Raise`를 기반으로 한 DSL**을 사용하면 보통 더 깔끔한 코드를 작성할 수 있습니다.

이 가이드에서는 `Either` 스타일의 일반적인 패턴들과 이것들이 `Raise`로 어떻게 변환되는지 설명합니다.

---

## 사전 준비: 예제 함수들

다음 설명에서는 아래 함수들이 정의되어 있다고 가정합니다:
```kotlin
fun f(n: Int): Either<Error, String>
fun g(s: String): Either<Error, Thing>
fun h(s: String): Either<Boo, Thing>
fun Thing.summarize(): String
```

---

## 순차적 합성 (Sequential Composition)

에러가 발생할 수 있는 계산들을 합성하는 기본 방법은 각각을 순차적으로 실행하고, 에러가 나타나면 조기 종료하는 것입니다.

### Either 스타일 (flatMap/map 사용)
```kotlin
fun foo(n: Int): Either<Error, String> =
    f(n).flatMap { s -> 
        g(s).map { t -> 
            t.summarize() 
        } 
    }
```

- `flatMap`: 다음 계산도 실패할 수 있을 때 사용
- `map`: 결과에 순수 함수를 적용할 때 사용

### Raise DSL 스타일 (권장)
```kotlin
fun foo(n: Int): Either<Error, String> = either {
    val s = f(n).bind()
    val t = g(s).bind()
    t.summarize()
}
```

변환의 핵심 포인트:

1. **`either` 빌더**: 함수 시작 부분에 위치하며, 두 가지 역할을 합니다:
    - 함수의 반환 타입이 `Either`임을 나타냄
    - Raise DSL을 사용할 수 있는 새로운 스코프 생성

2. **`bind()` 호출**: 다른 계산에서 나온 `Either` 값에 대해 `bind()`를 호출합니다:
    - 에러인 경우: 현재 계산을 **즉시 종료** (short-circuit)
    - 성공인 경우: 내부 값을 꺼내서 **정상 실행 계속**

한 줄로도 작성 가능합니다:
```kotlin
fun foo(n: Int): Either<Error, String> = either {
    g(f(n).bind()).bind().summarize()
}
```

### 다양한 빌더들

Arrow는 반환 타입에 따라 다양한 빌더를 제공합니다:

| 빌더 | 반환 타입 | 용도 |
|------|----------|------|
| `either { }` | `Either<E, A>` | 명시적 에러 타입이 필요할 때 |
| `option { }` | `Option<A>` | 값이 있거나 없는 경우 |
| `result { }` | `Result<A>` | Kotlin 표준 Result 사용 시 |

어떤 빌더를 선택하든, 실패 가능한 단계에서는 항상 `bind()`를 사용합니다.

---

## "Raise DSL"이라고 부르는 이유

코드에서 `Raise`라는 단어를 직접 사용하지 않는데 왜 "Raise DSL"이라고 할까요?

`either` 선언의 타입을 살펴보면:
```kotlin
fun <E, A> either(block: Raise<E>.() -> A): Either<E, A>
```

`Raise<E>`가 함수 타입의 앞에 나타나는 것(확장 리시버)은 `Raise<E>` 인터페이스의 모든 함수가 `block` 스코프 내에서 암묵적으로 사용 가능함을 의미합니다.

**`Raise<E>` 인터페이스에 포함된 함수들:**
- `bind()` - Either 값에서 성공 값 추출
- `raise()` - 에러 발생
- `ensure()` - 조건 확인
- `mapOrAccumulate()` - 에러 누적

---

## 논리적 에러로 반환하기

### Either 스타일의 에러 생성
```kotlin
// Left로 에러 생성
Either.Left(Error.NegativeInput)
```

### Raise DSL 스타일 (권장)
```kotlin
fun fooThatRaises(n: Int): Either<Error, String> = either {
    if (n < 0) raise(Error.NegativeInput)
    val s = f(n).bind()
    val t = g(s).bind()
    t.summarize()
}
```

`raise()`를 호출하면 현재 블록의 실행이 **즉시 종료**됩니다.

예시: `fooThatRaises(-1)` 호출 시
→ `raise(Error.NegativeInput)` 실행
→ 결과: `Either.Left(Error.NegativeInput)`

> **예외(Exception)와의 차이점**
>
> `Raise`의 전파와 조기 반환이 예외와 비슷해 보이지만, 목적이 다릅니다:
>
> | 구분 | Typed Errors (Raise) | Exceptions |
> |------|---------------------|------------|
> | 용도 | 논리적 에러 (도메인 모델에 속하는 문제) | 예외적 상황 (복구하기 어려운 상황) |
> | 예시 | "데이터베이스에서 사용자를 찾을 수 없음" | "데이터베이스 연결이 갑자기 끊김" |
> | 처리 | 명시적 타입으로 처리 | try-catch 또는 resilience 패턴 |

### ensure() 유틸리티 함수

조건 확인 후 실패 시 raise하는 패턴은 매우 흔해서, Arrow는 `ensure()` 함수를 제공합니다:
```kotlin
fun fooThatRaises(n: Int): Either<Error, String> = either {
    ensure(n >= 0) { Error.NegativeInput }  // 조건이 false면 에러 raise
    val s = f(n).bind()
    val t = g(s).bind()
    t.summarize()
}
```

> **추가 유틸리티 함수들**
> ```kotlin
> // ensureNotNull - null이면 에러
> ensureNotNull(value) { Error.NullValue }
> 
> // catch - 예외를 typed error로 변환
> catch({ riskyOperation() }) { e -> Error.FromException(e) }
> ```

---

## 에러 값 변환하기

`f`와 `g`가 같은 `Error` 타입을 공유하기 때문에 앞선 코드가 컴파일됩니다.
모든 `Raise` 스코프에서는 **하나의 에러 타입**만 raise하거나 bind할 수 있습니다.

에러 타입이 다른 경우 변환이 필요합니다.

### Either 스타일 (mapLeft 사용)
```kotlin
fun bar(n: Int): Either<Error, String> =
    f(n).flatMap { s ->
        h(s).mapLeft { it.toString() }.map { t -> 
            t.summarize() 
        }
    }
```

### Raise DSL 스타일 (withError 사용)
```kotlin
fun bar(n: Int): Either<Error, String> = either {
    val s = f(n).bind()
    val t = withError({ boo -> boo.toError() }) {
        h(s).bind()
    }
    t.summarize()
}
```

`withError`는 새로운 스코프를 생성하며, 내부 스코프의 에러 타입을 외부 스코프의 에러 타입으로 변환하는 방법을 첫 번째 인자로 받습니다.

---

## 여러 개의 실패 처리하기

여러 값에 대해 실패 가능한 계산을 실행해야 할 때가 있습니다.

### Either 스타일 (traverse 사용)
```kotlin
fun foos(xs: List<Int>) = xs.traverse { foo(it) }
```

### Raise DSL 스타일 (권장)
```kotlin
fun foos(xs: List<Int>) = either {
    xs.map { foo(it).bind() }
    // 또는
    xs.map { foo(it) }.bindAll()
}
```

> **핵심 장점**: 특별한 effectful 조합자가 필요 없습니다!
>
> 일반 컬렉션 함수(`map`, `filter`, `forEach` 등)를 제한 없이 사용할 수 있습니다.

### 에러 누적 (Error Accumulation)

기본적으로 `Raise`는 **fail-first** 방식으로 동작합니다: 실패가 발생하면 전체 블록이 즉시 중단됩니다.

컬렉션을 처리할 때 첫 번째 에러만 보고되는 것이 아니라 **모든 에러를 모아야** 한다면 `mapOrAccumulate`를 사용하세요:
```kotlin
fun validateAll(xs: List<Int>) = either {
    xs.mapOrAccumulate { item ->
        validate(item).bind()
    }
}
```

> **사용 사례**: 폼 유효성 검사에서 모든 필드의 에러를 한 번에 보여주고 싶을 때 유용합니다.

---

## Either와 bind 없이 사용하기 (Pure Raise 스타일)

지금까지는 `Either` 계산들을 조합하여 또 다른 `Either`를 만들었습니다.
`Raise`를 전면적으로 사용하면 `bind()`도 제거할 수 있습니다!

### 함수 시그니처 변경

**기존 (Either 스타일):**
```kotlin
fun f(n: Int): Either<Error, String>
fun g(s: String): Either<Error, Thing>
fun h(s: String): Either<Boo, Thing>
```

**변경 후 (Pure Raise 스타일):**
```kotlin
fun Raise<Error>.f(n: Int): String
fun Raise<Error>.g(s: String): Thing
fun Raise<Boo>.h(s: String): Thing
fun Thing.summarize(): String
```

에러 타입이 반환 타입을 감싸는 대신, **확장 리시버**로 나타납니다.

### 결과: 더 깔끔한 코드
```kotlin
fun Raise<Error>.bar(n: Int): String {
    val s = f(n)  // bind() 불필요!
    val t = withError({ boo -> boo.toError() }) {
        h(s)
    }
    return t.summarize()
}
```

> **권장 사항**: 특히 코드의 내부(non-public) 부분에서는 이 스타일을 권장합니다.
>
> **장점:**
> - 코드가 더 깔끔해짐
> - `Right`와 `Left` 값을 감싸고 푸는 오버헤드 없음
> - 일반 Kotlin 코드처럼 읽힘

---

## 현재 제약사항: 다중 리시버

현재 Kotlin 언어는 **두 개 이상의 리시버를 허용하지 않습니다**.

따라서 이런 함수를:
```kotlin
fun Thing.problematic(): Either<Error, String>
```

`Raise<Error>`를 리시버로 가진 버전으로 쉽게 변환할 수 없습니다.

### 임시 해결책

원래 리시버를 인자로 이동시킵니다:
```kotlin
fun Raise<Error>.problematic(thing: Thing): String
```

> Kotlin의 [Context Parameters](https://github.com/Kotlin/KEEP/issues/367) 제안이 채택되면 이 제약이 해소될 예정입니다.

