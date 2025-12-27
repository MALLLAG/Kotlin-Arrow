# 유효성 검사 (Validation)

이 튜토리얼은 도메인 유효성 검사를 구현하기 위해 타입화된 에러를 사용하는 구체적인 예제를 보여줍니다.

## 도메인 정의

먼저 다음과 같은 도메인으로 시작합니다:
```kotlin
data class Author(val name: String)
data class Book(val title: String, val authors: NonEmptyList<Author>)
```

## 유효성 검사 규칙

이 도메인에 대해 다음 규칙을 구현하고, **가능한 한 많은 에러를 누적**하고자 합니다:

1. 주어진 제목(title)이 비어있으면 안 됨
2. 저자(authors) 목록이 비어있으면 안 됨
3. 저자 이름 중 어느 것도 비어있으면 안 됨

> 💡 **"Parse, don't validate" 접근법**:
> 우리는 유효성 검사에 대해 "파싱하라, 유효성 검사하지 마라(parse, don't validate)" 접근법을 따를 것입니다. 간단히 말해, 잠재적으로 잘못된 클래스 인스턴스를 먼저 만드는 대신, **컴포넌트가 모든 제약 조건을 충족할 때만** 인스턴스를 생성합니다.
>
> ```kotlin
> // ❌ 전통적인 방식 - 유효성 검사
> val book = Book(title, authors)  // 일단 생성
> if (!book.isValid()) { ... }     // 나중에 검사
>
> // ✅ Arrow 방식 - 파싱
> val book: Either<Errors, Book> = Book(title, authors)  // 유효할 때만 생성됨
> ```

---

## 스마트 생성자 (Smart Constructors)

`Author` 클래스는 생성자를 노출하므로, 사용자가 잘못된 값으로 생성하는 것을 막을 수 없습니다. 생성자에 `require`를 도입할 수 있지만, 대신 **타입화된 에러 메커니즘**을 사용하는 것을 선호합니다.

### 패턴: private 생성자 + invoke 연산자

이 경우 일반적인 패턴은 생성자를 숨기고, **companion object 내에 `invoke` 연산자를 추가**하여 스마트 생성자를 제공하는 것입니다.
```kotlin
object EmptyAuthorName

data class Author private constructor(val name: String) {
    companion object {
        operator fun invoke(name: String): Either<EmptyAuthorName, Author> = TODO()
    }
}
```

이렇게 하면 이 클래스의 사용자는 여전히 `Author("me")`를 사용하여 새 이름을 생성합니다. 생성자를 사용하는 것과 같은 방식이지만, 실제로는 우리의 `invoke` 함수가 호출됩니다. 이를 통해 타입을 `Either`로 정제하여 에러를 반환할 수 있습니다.

> 💡 **왜 `invoke` 연산자인가?**
>
> Kotlin에서 `invoke` 연산자를 정의하면 객체를 함수처럼 호출할 수 있습니다:
> ```kotlin
> // 이 두 호출은 동일합니다
> Author.invoke("me")
> Author("me")  // invoke가 자동으로 호출됨
> ```
>
> 사용자 입장에서는 일반 생성자처럼 보이지만, 내부적으로는 유효성 검사를 수행합니다!

### 구현

`either` 연산 블록을 사용하고, `ensure`로 제약 조건 #3을 설명합니다:
```kotlin
data class Author private constructor(val name: String) {
    companion object {
        operator fun invoke(name: String): Either<EmptyAuthorName, Author> = either {
            ensure(name.isNotEmpty()) { EmptyAuthorName }
            Author(name)
        }
    }
}
```

> 💡 **코드 흐름 이해하기**:
> ```kotlin
> Author("")      // Either.Left(EmptyAuthorName)
> Author("Alice") // Either.Right(Author("Alice"))
> ```
>
> 빈 이름으로는 절대 `Author` 인스턴스를 생성할 수 없습니다!

---

## Fail-first vs 누적 (Accumulation)

`Book`에 대해서도 스마트 생성자를 도입하는 유사한 접근법을 사용할 것입니다. 하지만 여러 가지 다른 에러가 있으므로, 이를 **sealed 계층**으로 정의합니다.

```kotlin
sealed interface BookValidationError
object EmptyTitle : BookValidationError
object NoAuthors : BookValidationError
data class EmptyAuthor(val index: Int) : BookValidationError

data class Book private constructor(
    val title: String, 
    val authors: NonEmptyList<Author>
) {
    companion object {
        operator fun invoke(
            title: String, 
            authors: Iterable<String>
        ): Either<BookValidationError, Book> = TODO()
    }
}
```

### 첫 번째 시도: Fail-first 방식

잠시 각 저자의 유효성 검사는 잊고, 제목과 저자 목록에 대한 빈 값 검사만 구현해 보겠습니다.

> 💡 `ensureNotNull`을 사용하면 검사와 `NonEmptyList`로의 변환을 한 번에 수행합니다.
```kotlin
data class Book private constructor(
    val title: String, 
    val authors: NonEmptyList<Author>
) {
    companion object {
        operator fun invoke(
            title: String, 
            authors: Iterable<String>
        ): Either<BookValidationError, Book> = either {
            ensure(title.isNotEmpty()) { EmptyTitle }
            ensureNotNull(authors.toNonEmptyListOrNull()) { NoAuthors }
            Book(title, TODO())
        }
    }
}
```

### 문제점

이 코드에는 문제가 있습니다: Book의 데이터에 두 가지 문제가 있어도 **하나의 에러만 반환**합니다. 우리는 사용자에게 가능한 한 많은 정보를 돌려줄 수 있도록 **누적 접근법**을 사용하고 싶습니다.
```kotlin
// 현재 동작
Book("", emptyList())  // Either.Left(EmptyTitle) - 첫 번째 에러만!

// 원하는 동작
Book("", emptyList())  // Either.Left(listOf(EmptyTitle, NoAuthors)) - 모든 에러!
```

### 해결책: zipOrAccumulate 사용

코드에 두 가지 변경이 필요합니다:

1. 결과 타입이 이제 **문제들의 NonEmptyList**
2. 다른 유효성 검사들을 **`zipOrAccumulate`로 래핑**
```kotlin
data class Book private constructor(
    val title: String, 
    val authors: NonEmptyList<Author>
) {
    companion object {
        operator fun invoke(
            title: String, 
            authors: Iterable<String>
        ): Either<NonEmptyList<BookValidationError>, Book> = either {
            zipOrAccumulate(
                { ensure(title.isNotEmpty()) { EmptyTitle } },
                { ensureNotNull(authors.toNonEmptyListOrNull()) { NoAuthors } }
            ) { _, _ -> Unit }
            Book(title, TODO())
        }
    }
}
```

`zipOrAccumulate`의 각 인수의 결과는 후행 람다에서 사용할 수 있습니다. 이 경우에는 사용하지 않습니다: 제목은 이미 사용 가능하고, 저자 목록은 아직 `String`에서 `Author`로의 변환을 수행해야 합니다.

> 💡 **zipOrAccumulate의 인수**:
>
> `zipOrAccumulate`의 마지막 인수를 제외한 모든 인수는 출력을 집계하면서 실행하려는 다른 유효성 검사를 나타냅니다. 이러한 인수는 종종 `{ 중괄호 }`로 감싸진 블록이며, 대부분의 Kotlin 개발자에게는 약간 익숙하지 않을 수 있습니다.
>
> ```kotlin
> zipOrAccumulate(
>     { /* 검증 1 */ },
>     { /* 검증 2 */ },
>     { /* 검증 3 */ }
> ) { result1, result2, result3 ->
>     // 모든 검증이 성공하면 여기서 결과를 조합
> }
> ```

---

## 리스트 유효성 검사

다음 단계는 주어진 `authors`(문자열 리스트)를 `Author` 리스트로 변환하는 것입니다. 스마트 생성자를 실행해야 하지만, 동시에 잠재적인 문제도 누적해야 합니다.

이것은 저자 검사와 관련되므로, 두 번째 유효성 검사의 일부로 포함하겠습니다.
```kotlin
data class Book private constructor(
    val title: String, 
    val authors: NonEmptyList<Author>
) {
    companion object {
        operator fun invoke(
            title: String, 
            authors: Iterable<String>
        ): Either<NonEmptyList<BookValidationError>, Book> = either {
            zipOrAccumulate(
                { ensure(title.isNotEmpty()) { EmptyTitle } },
                { 
                    val validatedAuthors = mapOrAccumulate(authors.withIndex()) { nameAndIx ->
                        Author(nameAndIx.value)
                            .recover { _ -> raise(EmptyAuthor(nameAndIx.index)) }
                            .bind()
                    }
                    ensureNotNull(validatedAuthors.toNonEmptyListOrNull()) { NoAuthors }
                }
            ) { _, authorsNel -> 
                Book(title, authorsNel) 
            }
        }
    }
}
```

### 코드 분석

이 추가 검사는 꽤 복잡하므로, 단계별로 풀어보겠습니다:

#### 1. withIndex() 사용
```kotlin
authors.withIndex()
```

값과 함께 해당 값이 위치한 인덱스를 포함하는 iterable을 생성합니다. 올바른 `EmptyAuthor` 에러 값을 만들기 위해 필요합니다.
```kotlin
listOf("Alice", "", "Bob").withIndex()
// IndexedValue(0, "Alice"), IndexedValue(1, ""), IndexedValue(2, "Bob")
```

#### 2. mapOrAccumulate로 에러 누적
```kotlin
mapOrAccumulate(authors.withIndex()) { nameAndIx ->
    // 각 요소에 대한 유효성 검사, 에러 누적
}
```

컬렉션의 요소에 대해 유효성 검사를 수행하고 각 가능한 에러를 누적하고 싶다고 명시합니다.

#### 3. recover로 에러 타입 변환
```kotlin
Author(nameAndIx.value)
    .recover { _ -> raise(EmptyAuthor(nameAndIx.index)) }
```

`Author(it.value)` 호출은 잘못된 에러 타입(`EmptyAuthor` 대신 `EmptyAuthorName`)을 가진 `Either`를 반환합니다. 이 값을 변환하기 위해 `recover` 확장 함수를 사용합니다.

> 💡 **recover vs mapLeft**:
>
> 다른 가능성은 `mapLeft { EmptyAuthor(it.index) }`를 사용하는 것입니다.
>
> | `recover` | `mapLeft` |
> |-----------|-----------|
> | 어떤 타입화된 에러 연산도 사용 가능 | 에러 값만 변환 |
> | `raise`를 호출할 수 있음 | 단순 매핑만 가능 |
>
> ```kotlin
> // mapLeft 사용
> Author(name).mapLeft { EmptyAuthor(index) }
> 
> // recover 사용 - 더 유연함
> Author(name).recover { _ -> 
>     // 여기서 다른 로직을 수행할 수 있음
>     raise(EmptyAuthor(index)) 
> }
> ```

#### 4. bind()로 Either를 연산 블록에 임베딩
```kotlin
.bind()
```

`either`(또는 다른 `Raise` 블록) 내에서 `Either` 타입의 값을 사용할 때마다 이러한 호출이 필요합니다.

#### 5. 결과 사용

매핑의 결과는 `List<Author>`이며, 이제 최종 `Book`을 만드는 데 사용할 수 있습니다. 이 값은 `zipOrAccumulate`의 마지막 람다에서 사용할 수 있으며, 위 코드에서 `validatedAuthors`라고 불렀습니다.

> 💡 **전체 흐름 요약**:
> ```
> authors: Iterable<String>
>     ↓ withIndex()
> IndexedValue<String>들
>     ↓ mapOrAccumulate
> 각 요소에 대해:
>     Author(name) → Either<EmptyAuthorName, Author>
>         ↓ recover
>     Either<EmptyAuthor, Author>
>         ↓ bind()
>     Author (또는 에러 누적)
>     ↓
> List<Author> 또는 NonEmptyList<BookValidationError>
>     ↓ toNonEmptyListOrNull() + ensureNotNull
> NonEmptyList<Author> 또는 NoAuthors 에러
> ```

---

## map + 누적의 변형들

위 코드에서 각 요소 처리 중 발생한 에러를 누적하면서 리스트를 매핑하는 섹션은 여러 가지 다른 방식으로 작성할 수 있습니다.

### 방법 1: Raise 내의 mapOrAccumulate
```kotlin
val validatedAuthors = mapOrAccumulate(authors.withIndex()) { nameAndIx ->
    Author(nameAndIx.value)
        .mapLeft { EmptyAuthor(nameAndIx.index) }
        .bind()
}
```

이 버전은 `Raise`에 있고 작업할 컬렉션을 첫 번째 인수로 받는 `mapOrAccumulate` 변형을 사용합니다. 이 변형은 블록 내에서 `Raise`를 제공하므로(따라서 `.bind()` 호출이 필요), 에러가 발견되면 자동으로 raise합니다.

### 방법 2: map + bindAll()

위 코드를 작성하는 또 다른 방법은 `map`을 사용하여 `Either` 리스트를 만들고, 맨 마지막에 `.bindAll()`을 사용하는 것입니다.
```kotlin
val validatedAuthors = authors.withIndex().map { nameAndIx ->
    Author(nameAndIx.value)
        .mapLeft { EmptyAuthor(nameAndIx.index) }
}.bindAll()
```

유효성 검사가 래퍼 타입을 사용할 때(여기서처럼) 중간 `.bind()`를 호출할 필요가 없으므로 이 방식이 종종 더 간단한 코드가 됩니다.

> 💡 **어떤 방식을 선택해야 할까?**
>
> | 방식 | 장점 | 단점 |
> |------|------|------|
> | `mapOrAccumulate` + `bind()` | Raise 컨텍스트 내에서 일관성 | 각 요소마다 `bind()` 필요 |
> | `map` + `bindAll()` | 래퍼 타입 사용 시 더 간결 | 두 단계 분리 |
>
> 각 요소를 유효성 검사하는 함수가 **부수 효과를 수행하지 않는다면**, 이러한 접근법들은 동등합니다.

---

## 전체 예제 코드
```kotlin
// 에러 타입 정의
object EmptyAuthorName

sealed interface BookValidationError
object EmptyTitle : BookValidationError
object NoAuthors : BookValidationError
data class EmptyAuthor(val index: Int) : BookValidationError

// Author 스마트 생성자
data class Author private constructor(val name: String) {
    companion object {
        operator fun invoke(name: String): Either<EmptyAuthorName, Author> = either {
            ensure(name.isNotEmpty()) { EmptyAuthorName }
            Author(name)
        }
    }
}

// Book 스마트 생성자 (에러 누적)
data class Book private constructor(
    val title: String, 
    val authors: NonEmptyList<Author>
) {
    companion object {
        operator fun invoke(
            title: String, 
            authors: Iterable<String>
        ): Either<NonEmptyList<BookValidationError>, Book> = either {
            zipOrAccumulate(
                { ensure(title.isNotEmpty()) { EmptyTitle } },
                { 
                    val validatedAuthors = authors.withIndex().map { nameAndIx ->
                        Author(nameAndIx.value)
                            .mapLeft { EmptyAuthor(nameAndIx.index) }
                    }.bindAll()
                    ensureNotNull(validatedAuthors.toNonEmptyListOrNull()) { NoAuthors }
                }
            ) { _, authorsNel -> 
                Book(title, authorsNel) 
            }
        }
    }
}
```

### 사용 예제
```kotlin
fun main() {
    // 성공 케이스
    val validBook = Book("Kotlin in Action", listOf("Alice", "Bob"))
    println(validBook) 
    // Either.Right(Book(title=Kotlin in Action, authors=[Author(Alice), Author(Bob)]))
    
    // 실패 케이스 - 모든 에러가 누적됨
    val invalidBook = Book("", listOf("", "Bob", ""))
    println(invalidBook)
    // Either.Left([EmptyTitle, EmptyAuthor(index=0), EmptyAuthor(index=2)])
    
    // 저자 없음
    val noAuthorsBook = Book("Some Title", emptyList())
    println(noAuthorsBook)
    // Either.Left([NoAuthors])
}
```
