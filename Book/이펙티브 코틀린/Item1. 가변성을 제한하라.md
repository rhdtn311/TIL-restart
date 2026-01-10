# 목차

1. 개요
2. 상태(State) 관리와 가변성의 문제점
3. 가변성 제한 방법
   3.1. 읽기 전용 프로퍼티 (`val`)
   3.2. 가변 컬렉션과 읽기 전용 컬렉션의 분리
   3.3. 데이터 클래스의 `copy` 메서드 활용
4. 가변 지점(Mutation Point)의 위치 선정
5. 실무 적용 사례
6. 요약

---

> 이 문서는 사용자의 초안을 바탕으로 Gemini가 체계적으로 구조화하고 내용을 다듬어 작성했다.

---

### 1. 개요

코틀린 개발 시 객체가 상태(State)를 갖도록 설계하는 것은 흔한 패턴이다. 그러나 무분별한 상태 변경은 코드의 복잡성을 높이고 유지보수를 어렵게 만든다. 이 문서에서는 가변성(Mutability)으로 인해 발생할 수 있는 문제점을 분석하고, 코틀린에서 이를 효과적으로 제한하는 기법을 다룬다.

---

### 2. 상태(State) 관리와 가변성의 문제점

아래 코드는 객체가 상태를 가질 때의 전형적인 예시다.

```kotlin
class BankAccount {
    var balance = 0.0
        private set

    fun deposit(depositAmount: Double) {
        balance += depositAmount
    }
}

```

위와 같이 객체 내부의 상태(`balance`)가 변경 가능할 때, 다음과 같은 문제들이 발생한다.

1. **디버깅 및 코드 이해의 어려움**
* 상태 변경이 빈번하면 데이터의 변화 과정을 추적하기 어렵다.
* 특정 시점의 상태를 예측하기 어려워 예기치 못한 버그가 발생할 수 있다.


2. **동기화(Synchronization) 문제**
* 멀티스레드 환경에서 여러 스레드가 동시에 상태를 변경할 경우 경합 조건(Race Condition)이 발생할 수 있다.
* 적절한 동기화 처리가 없으면 데이터 무결성이 깨진다.


3. **테스트의 복잡성 증가**
* 가능한 모든 상태 조합을 테스트해야 하므로, 변경 가능한 요소가 많을수록 테스트 케이스가 기하급수적으로 늘어난다.



---

### 3. 가변성 제한 방법

코틀린은 언어 차원에서 가변성을 제한할 수 있는 다양한 도구를 제공한다.

* 읽기 전용 프로퍼티 `val`
* 가변 컬렉션과 읽기 전용 컬렉션의 분리
* 데이터 클래스의 `copy` 메서드

#### 3.1. 읽기 전용 프로퍼티 (`val`)

`val` 키워드를 사용하면 읽기 전용 프로퍼티를 선언할 수 있다. 이는 일반적인 재할당을 막아주어 값의 안정성을 높인다.

```kotlin
val a = 10
a = 20 // 컴파일 오류 발생

```

**주의: `val`이 완전한 불변(Immutable)을 의미하지는 않는다.**
`val`은 참조 자체를 변경할 수 없다는 의미일 뿐, 참조된 객체의 내부 상태는 변경될 수 있다.

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)
print(list) // 1, 2, 3, 4

```

또한, **사용자 정의 게터(Custom Getter)**를 사용할 경우 호출 시점마다 다른 값을 반환할 수 있다.

```kotlin
var firstName = "TaeHyun"
var lastName = "Kong"
val fullName: String
    get() = "$lastName $firstName"

fun main() {
    println(fullName)  // Kong TaeHyun
    firstName = "TH"
    println(fullName)  // Kong TH
}

```

**`val`의 특징 및 제약 사항:**

* **오버라이드 가능:** 인터페이스에서 `val`로 정의되었더라도 구현체에서 `var`로 오버라이드할 수 있다.
* **스마트 캐스트 제한:** 사용자 정의 게터가 있는 `val`은 호출 시점에 값이 달라질 수 있으므로 스마트 캐스트가 불가능하다. 반면, 지역 변수가 아니더라도 `final`이고 커스텀 게터가 없는 `val`은 스마트 캐스트가 가능하다.

#### 3.2. 가변 컬렉션과 읽기 전용 컬렉션의 분리

코틀린은 컬렉션 인터페이스를 **읽기 전용(`List`, `Set` 등)**과 **가변(`MutableList`, `MutableSet` 등)**으로 명확히 구분한다.

내부적으로는 가변 컬렉션(예: `ArrayList`)을 사용하더라도, 외부에는 읽기 전용 인터페이스(`Iterable`, `List`)로 노출하여 불변성을 유지하는 패턴을 자주 사용한다.

```kotlin
inline fun <T, R> Iterable<T>.map(
    transformation: (T) -> R
): List<R> {
    val list = ArrayList<R>() // 내부에서는 가변 리스트 사용
    for (elem in this) {
        list.add(transformation(elem))
    }
    return list // 반환 시에는 읽기 전용 인터페이스로 반환
}

```

> **주의: 다운캐스팅(Downcasting)의 위험성**
> 코틀린의 `List`는 런타임 시 자바의 `java.util.List`로 컴파일된다. 자바는 읽기 전용과 가변 리스트를 타입으로 구분하지 않으므로, 런타임에는 다운캐스팅이 성공할 수 있다. 하지만 실제 객체가 불변 리스트(`Arrays.asList` 등)인 경우 데이터 수정 시 예외가 발생한다.

```kotlin
fun main() {
    val list = listOf(1, 2, 3, 4, 5)

    // 런타임에는 캐스팅이 성공함 (Java의 List 인터페이스로 취급)
    if (list is MutableList<Int>) {
        // 실제 객체는 불변이므로 UnsupportedOperationException 발생
        list.add(6) 
    }
}

```

따라서 리스트를 변경해야 한다면 다운캐스팅 대신 `.toMutableList()`를 사용하여 새로운 복사본을 생성해야 한다.

#### 3.3. 데이터 클래스의 `copy` 메서드 활용

불변 객체의 단점은 값을 수정할 수 없다는 것이다. 이를 해결하기 위해 객체를 수정하는 대신, 변경된 값을 가진 **새로운 객체를 생성**하는 방식을 사용한다. 코틀린의 `data class`는 `copy` 메서드를 자동으로 제공하여 이를 간편하게 지원한다.

```kotlin
data class User(val name: String, val age: Int)

fun main() {
    val youngUser = User(name = "Kong", age = 30)
    // 기존 객체는 유지하고, 일부 속성만 변경한 새 객체 생성
    val oldUser = youngUser.copy(name = "Taehyun", age = 31) 
    
    println(oldUser)  // User(name=Taehyun, age=31)  
}

```

이 방식은 데이터의 불변성을 유지하면서도 상태 변경과 유사한 효과를 낼 수 있어, 스레드 안전성을 확보하는 데 유리하다.

---

### 4. 가변 지점(Mutation Point)의 위치 선정

변경 가능한 리스트가 필요할 때, 가변 지점을 어디에 둘지 결정해야 한다.

1. **내부 가변성:** 리스트 자체가 변경 가능 (`val list: MutableList`)
2. **프로퍼티 가변성:** 프로퍼티 자체가 변경 가능 (`var list: List`)

```kotlin
// 1. 구체적인 리스트 구현 내부에 변경 가능 지점이 있음
val list1: MutableList<Int> = mutableListOf() 

// 2. 프로퍼티 자체가 변경 가능 지점임
var list2: List<Int> = listOf() 

```

두 방식 모두 데이터 추가가 가능하다.

```kotlin
list1.add(1)        // 리스트 내부 상태 변경
list2 = list2 + 1   // 새로운 리스트를 생성하여 프로퍼티 재할당

```

**멀티스레드 환경에서의 안전성 비교:**

* **`list1` (MutableList):** 리스트 내부의 구현에 의존한다. 표준 `ArrayList` 등은 동기화 처리가 되어 있지 않으므로 멀티스레드 환경에서 안전하지 않다.
* **`list2` (var List):** 프로퍼티에 대한 재할당이 발생한다. 이 경우 사용자 정의 세터나 델리게이트(Delegate)를 사용하여 동기화 로직을 추가할 수 있다. 따라서 동기화 제어 측면에서 두 번째 방식(`var` + 읽기 전용 `List`)이 더 유연하고 안전하게 관리될 수 있다.

---

### 5. 실무 적용 사례

안드로이드(Android) 개발이나 서버 사이드 개발에서 상태 관리는 매우 중요하다. **Backing Property** 패턴은 가변성을 제한하는 대표적인 실무 사례다.

**ViewModel에서의 상태 노출 패턴:**

```kotlin
class MainViewModel : ViewModel() {
    // 내부에서는 수정 가능한 MutableStateFlow 사용 (쓰기 가능)
    private val _uiState = MutableStateFlow(UiState())
    
    // 외부에는 읽기 전용 StateFlow로 노출 (읽기 전용)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun updateData() {
        // 내부에서는 상태 변경 가능
        _uiState.update { it.copy(isLoading = true) }
    }
}

```

* **내부 (`_uiState`):** 클래스 내부 로직에서는 상태를 변경해야 하므로 `Mutable` 타입을 사용한다.
* **외부 (`uiState`):** UI 레이어 등 외부에서는 상태를 구독(Observe)만 하고 직접 변경하지 못하도록 읽기 전용 인터페이스로 노출한다.
* 이 패턴을 통해 데이터의 흐름을 단방향으로 유지하고, 외부에서의 무분별한 상태 변경을 방지할 수 있다.

---

### 6. 요약

가변성을 제한하는 것은 시스템의 안정성과 예측 가능성을 높이는 핵심 요소다.

1. **`val` 사용:** 기본적으로 읽기 전용 프로퍼티를 사용하여 재할당을 막는다.
2. **불변 컬렉션 활용:** 가변 컬렉션(`MutableList`)보다는 읽기 전용 컬렉션(`List`)을 사용하고, 필요한 경우 `copy`를 통해 새로운 객체를 생성한다.
3. **가변 지점 제어:** 상태 변경이 꼭 필요한 경우, 변경 지점을 명확히 하고 동기화가 용이한 방식(`var` + 불변 데이터)을 고려한다.

이러한 원칙을 준수하면 디버깅이 쉽고, 멀티스레드 환경에서 안전하며, 테스트하기 좋은 코드를 작성할 수 있다.