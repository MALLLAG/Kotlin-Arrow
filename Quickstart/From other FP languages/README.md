# 다른 함수형 프로그래밍 언어에서 Arrow로 전환하기

Arrow는 함수형 프로그래밍에서 큰 영향을 받았습니다. 이미 함수형 프로그래밍 개념에 익숙하다면, Arrow로의 전환은 즐거운 여정이 될 것입니다. 이 섹션에서는 다른 생태계와의 주요 차이점을 살펴봅니다.

## 영감을 준 언어들

**Scala**: Scala 표준 라이브러리에는 `Either`와 같이 Arrow가 제공하는 많은 타입들이 포함되어 있습니다. 또한 Typelevel 생태계와 같이 더 많은 함수형 기능을 제공하는 활발한 커뮤니티가 있습니다.

**Haskell**: Haskell은 순수 함수형 프로그래밍 언어의 대표적인 예로 여겨집니다. Arrow의 대부분의 유틸리티는 Haskell의 base 라이브러리에서 찾을 수 있습니다.

---

## 연산 블록 (Computation Blocks)

### Either에서 Raise로 전환하기

Arrow에는 에러를 감싸는 래퍼 타입(`Either` 등)에서 `Raise` DSL로 전환하는 것에 특화된 튜토리얼이 있습니다.

### for comprehension과 do notation

Scala와 Haskell 모두 `flatMap` 또는 `bind` 연산을 정의하는 타입에 대해 특별한 지원을 제공합니다. Scala에서는 `for comprehension`, Haskell에서는 `do notation`이라고 합니다. `Either`가 이러한 타입 중 하나이므로, `for`나 `do`를 사용하여 유효성 검사를 수행할 수 있습니다.

**Scala 예제:**
```scala
def mkPerson(name: String, age: Int): Either[Problem, Person] =
  for {
    name_ <- validName(name)
    age_  <- validAge(age)
  } yield Person(name_, age_)
```

**Haskell 예제:**
```haskell
mkPerson :: String -> Int -> Either Problem Person
mkPerson name age = do
  name_ <- validName name
  age_  <- validAge age
  pure (Person name_ age_)
```

Haskell에서는 `Applicative` 연산자를 사용하여 이 스타일에 더 가깝게 작성할 수 있습니다. 다만 코드가 순수한 형태와 다르게 보이는데, `(<$>)`와 `(<*>)`를 여기저기 뿌려야 하기 때문입니다.
```haskell
mkPerson name age = Person <$> validName name <*> validAge age
```

### Kotlin(Arrow)에서의 방식

Kotlin은 이러한 범용적인 구조를 제공하지 않습니다. 그러나 **Arrow는 에러 타입에 대해 유사한 문법을 제공합니다.**

주요 특징:
- `for` 대신 `either`, `result`, `nullable`을 사용하여 에러 타입으로 작업하겠다고 명시적으로 요청해야 합니다
- 이 함수들은 `arrow.core.raise` 패키지에 있습니다
- `<-`의 모든 사용은 `.bind()` 호출로 변환됩니다

**Kotlin(Arrow) 예제:**
```kotlin
fun mkPerson(name: String, age: Int): Either<Problem, Person> = either {
    val name_ = validName(name).bind()
    val age_ = validAge(age).bind()
    Person(name_, age_)
}
```

> 💡 **핵심 포인트**: `either { }` 블록 안에서 `.bind()`를 호출하면, `Either.Left`인 경우 즉시 블록을 빠져나가고, `Either.Right`인 경우 내부 값을 추출합니다. 이는 마치 try-catch처럼 동작하지만 타입 안전성을 보장합니다!

또한 `.bind()`의 결과는 일반 타입이므로, 원한다면 호출을 완전히 인라인할 수 있습니다. 이 스타일은 Haskell의 `Applicative` 연산자 사용과 매우 유사하지만, 연산자가 함수 수준이 아닌 인수 수준에 나타납니다.
```kotlin
fun mkPerson(name: String, age: Int): Either<Problem, Person> = either {
    Person(validName(name).bind(), validAge(age).bind())
}
```

> 💡 **실용적인 팁**: 위의 인라인 버전은 코드가 간결하지만, 디버깅 시에는 첫 번째 버전(변수를 사용하는 방식)이 더 유용할 수 있습니다. 중간 값을 확인하기 쉽기 때문이죠!

### zip이 필요 없는 이유

`for comprehension` 대신 래퍼 내부의 값들을 결합하기 위해 `zip` 같은 함수를 사용하는 것이 일반적입니다. Haskell에서는 종종 `(<$>)`와 `(<*>)` 형태를 취합니다. **Arrow에서는 동시성을 다룰 때를 제외하고는 블록을 사용하는 것을 선호합니다.**

> 💡 **왜 그럴까요?** Arrow의 `either { }` 블록은 순차적으로 실행되므로, 병렬 처리가 필요한 경우에만 `zip`을 사용합니다. 순차적인 검증에서는 블록이 더 읽기 쉽습니다!

### traverse가 필요 없는 이유

다른 FP 언어에서는 컬렉션의 각 요소에 효과적인(effectful) 연산을 적용하려면 `map`이 아닌 `traverse`라는 다른 함수를 사용해야 합니다. **Arrow에서는 이러한 분리가 존재하지 않습니다:** 블록 내에서 여러분이 익숙한 컬렉션 API의 동일한 함수들을 사용할 수 있습니다.

> 💡 **실용 예제**: Arrow에서는 이렇게 할 수 있습니다:
> ```kotlin
> either {
>     val results = items.map { item -> 
>         validateItem(item).bind() 
>     }
> }
> ```
> 별도의 `traverse` 함수 없이 일반 `map`과 `bind`로 해결됩니다!

---

## `suspend` vs `IO`

Arrow가 제공하는 부수 효과(side effects) 처리 유틸리티는 **코루틴**, 즉 `suspend`로 표시된 함수를 기반으로 합니다.

반면, Haskell은 부수 효과를 표시하기 위해 `IO`라는 특별한 래퍼 타입을 도입하며, Cats Effect 같은 인기 있는 Scala 라이브러리도 이 방식을 따릅니다.

> 💡 **Kotlin 개발자를 위한 설명**: `suspend` 함수는 Kotlin의 내장 기능으로, 비동기 코드를 동기 코드처럼 작성할 수 있게 해줍니다. Haskell의 `IO`나 Scala의 `IO` 모나드와 달리, Kotlin에서는 별도의 래퍼 타입 없이도 부수 효과를 안전하게 다룰 수 있습니다.

---

## 고차 타입 추상화 (Higher-kinded Abstractions)

Scala와 Haskell 모두 타입 생성자 수준에서 작동하는 추상화를 허용합니다. 예를 들어, 항상 `F<A>.flatMap(next: (A) -> F<B>): F<B>` 형태를 갖는 `flatMap` 같은 함수는 `Monad`라는 인터페이스/타입 클래스의 일부입니다.

**Kotlin은 이 기능을 제공하지 않지만**, Arrow는 일관성을 위해 명명 규칙을 따릅니다.

다음 목록은 Arrow의 함수 이름들을 Cats와 Haskell의 base 라이브러리의 추상화와 연관시킵니다:

| Arrow 함수 | 유래 | 설명 |
|-----------|------|------|
| `map` | Functor | 래퍼 내부 값을 변환 |
| `contramap` | Contravariant Functor | 역방향 변환 |
| `fold` | Foldable | 컬렉션을 단일 값으로 축소 |
| `zip` | Applicative (다른 언어에서는 `product` 또는 `(&&)`) | 여러 래퍼 값을 결합 |
| `traverse`, `sequence` | Traversable | **deprecated** - 일반 리스트 함수와 연산 블록으로 같은 동작 가능 |

> 💡 **Higher-Kinded Types(HKT)란?**:
> 간단히 말해, "타입의 타입"을 다루는 것입니다. `List<Int>`에서 `List`는 타입 생성자이고, `Int`를 받아 구체적인 타입을 만듭니다. HKT를 지원하면 이 `List` 자체를 추상화할 수 있습니다.
>
> Kotlin은 이를 직접 지원하지 않지만, Arrow는 다른 방법으로 비슷한 효과를 제공합니다.

### Semigroup과 Monoid

Arrow Core에는 `Semigroup`과 `Monoid`가 인터페이스로 포함되어 있습니다. 그러나 이들은 **deprecated**로 표시되어 있으며, Arrow 2.0에서 제거될 예정입니다.

`Semigroup`이나 `Monoid` 인수가 필요했던 함수들은 결합 함수와 해당하는 빈 요소를 받는 변형으로 대체되었습니다. 이 설계는 `kotlin.collections`의 `fold` 함수와 같이 Kotlin 생태계의 다른 부분과 더 잘 맞습니다.

> 💡 **왜 deprecated 되었을까?**
> Kotlin에서는 타입 클래스를 언어 수준에서 지원하지 않으므로, 인터페이스로 `Semigroup`과 `Monoid`를 표현하는 것이 자연스럽지 않습니다. 대신 람다를 직접 전달하는 방식이 더 Kotlin스럽고 실용적입니다.
>
> ```kotlin
> // 예전 방식 (deprecated)
> // listOf(1, 2, 3).combineAll(IntMonoid)
> 
> // 새로운 방식 (Kotlin스러운)
> listOf(1, 2, 3).fold(0) { acc, n -> acc + n }
> ```
