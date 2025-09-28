이 문서는 사용자의 초안을 바탕으로 Gemini가 체계적으로 구조화하고 내용을 다듬어 작성했습니다.

### 1. Bean Validation 개요

**Bean Validation**은 자바 애플리케이션에서 객체의 데이터 유효성을 검사하기 위한 표준 사양이다. 객체의 필드가 특정 규칙에 맞게 작성되었는지 어노테이션을 통해 간단하게 검증할 수 있는 기술 명세를 제공한다. 이는 **Jakarta Bean Validation**(과거 JSR-380) 사양에 정의되어 있다.

가장 널리 사용되는 구현체로는 **Hibernate Validator**가 있으며, 이는 ORM 프레임워크와는 무관한 독립적인 유효성 검증 라이브러리다.

---

### 2. 사용법

#### 2.1. 의존성 추가

Spring Boot 환경에서 Bean Validation을 사용하기 위해 다음 의존성을 추가한다.

> `spring-boot-starter-web` 의존성에 기본적으로 포함되어 있으나, 버전을 명확히 관리하고 싶을 경우 명시적으로 추가하는 것을 권장한다. 이 의존성을 추가하면 Spring Boot가 자동으로 **Hibernate Validator**를 프로젝트에 포함시킨다.

```kotlin
dependencies {
    // ... 다른 의존성들
    implementation("org.springframework.boot:spring-boot-starter-web")

    // Bean Validation을 위한 의존성
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

#### 2.2. 기본 사용법

의존성을 추가한 후, 검증이 필요한 DTO(Data Transfer Object) 객체의 필드에 기본 어노테이션을 적용하여 유효성 규칙을 정의한다.

- **DTO 필드에 어노테이션 적용**
    ```kotlin
    import jakarta.validation.constraints.Email
    import jakarta.validation.constraints.NotBlank
    import jakarta.validation.constraints.Size
    import jakarta.validation.constraints.Min
    
    data class UserCreateRequest(
        @field:NotBlank(message = "사용자 이름은 필수 입력 항목입니다.")
        @field:Size(min = 2, max = 10, message = "사용자 이름은 2자 이상 10자 이하로 입력해주세요.")
        val username: String,
    
        @field:NotBlank(message = "이메일은 필수 입력 항목입니다.")
        @field:Email(message = "올바른 이메일 형식이 아닙니다.")
        val email: String,
    
        @field:Min(value = 19, message = "나이는 19세 이상이어야 합니다.")
        val age: Int
    )
    ```

- Controller에서 @Valid 적용

  Controller의 메서드 파라미터에 @Valid 어노테이션을 추가하면, 해당 객체에 정의된 유효성 검증 규칙이 자동으로 실행된다.

    ```kotlin
    import com.example.validation.dto.UserCreateRequest
    import jakarta.validation.Valid
    import org.springframework.http.ResponseEntity
    import org.springframework.validation.BindingResult
    import org.springframework.web.bind.annotation.PostMapping
    import org.springframework.web.bind.annotation.RequestBody
    import org.springframework.web.bind.annotation.RequestMapping
    import org.springframework.web.bind.annotation.RestController
    
    @RestController
    @RequestMapping("/api/users")
    class UserController {
    
        @PostMapping
        fun createUser(
            @Valid @RequestBody userCreateRequest: UserCreateRequest,
            bindingResult: BindingResult // @Valid 파라미터 바로 뒤에 위치해야 한다.
        ): ResponseEntity<String> {
    
            // 1. 유효성 검증 실패 시 에러 처리
            if (bindingResult.hasErrors()) {
                val errorMessage = bindingResult.fieldError?.defaultMessage ?: "유효성 검증 실패"
                return ResponseEntity.badRequest().body(errorMessage)
            }
    
            // 2. 유효성 검증 성공 시 비즈니스 로직 처리
            println("요청 받은 데이터: $userCreateRequest")
            return ResponseEntity.ok("회원가입이 성공적으로 완료되었습니다.")
        }
    }
    ```

    - **`@Valid`**: Spring이 해당 어노테이션을 인지하고, `@RequestBody`로 매핑된 객체에 대해 유효성 검증을 실행한다.

    - **`BindingResult`**: 유효성 검증 결과를 담는 객체다. 검증 실패 시, 에러 정보가 이 객체에 담기며 예외가 발생하지 않는다. 만약 `BindingResult`가 없으면 검증 실패 시 `MethodArgumentNotValidException`이 발생한다.


#### 2.3. 커스텀 어노테이션 활용

표준 어노테이션으로 표현할 수 없는 복잡한 검증 규칙은 커스텀 어노테이션을 정의하여 구현할 수 있다. "전화번호는 반드시 하이픈(-)을 포함해야 한다"는 규칙을 예시로 한다.

1. **어노테이션 인터페이스 정의**

    ```kotlin
    import jakarta.validation.Constraint
    import jakarta.validation.Payload
    import kotlin.reflect.KClass
    
    @Target(AnnotationTarget.FIELD)
    @Retention(AnnotationRetention.RUNTIME)
    @Constraint(validatedBy = [PhoneNumberValidator::class]) // 검증 로직을 담은 클래스 지정
    annotation class PhoneNumber(
        val message: String = "전화번호 형식이 올바르지 않습니다. (하이픈 포함)",
        val groups: Array<KClass<*>> = [],
        val payload: Array<KClass<out Payload>> = []
    )
    ```

2. **유효성 검증기(Validator) 구현**
    ```kotlin
    import jakarta.validation.ConstraintValidator
    import jakarta.validation.ConstraintValidatorContext
    
    class PhoneNumberValidator : ConstraintValidator<PhoneNumber, String> {
        override fun isValid(value: String?, context: ConstraintValidatorContext?): Boolean {
            if (value.isNullOrBlank()) {
                return true // null 또는 빈 값은 검증 통과 (필수 여부는 @NotBlank 등으로 별도 처리)
            }
            // 하이픈(-) 포함 여부만 검증
            return value.contains("-")
        }
    }
    ```

3. **DTO에 적용**
    ```kotlin
    data class UserProfile(
        // ...
        @field:PhoneNumber
        val phoneNumber: String
    )
    ```


#### 2.4. 유효성 검증 그룹(Validation Groups)

하나의 DTO를 등록, 수정 등 여러 상황에서 사용하되, 상황별로 다른 검증 규칙을 적용해야 할 때 유용하다.

1. **그룹용 인터페이스 정의**

    ```kotlin
    interface ValidationGroups {
        interface Create
        interface Update
    }
    ```

2. **DTO 필드에 그룹 지정**

    ```kotlin
    import com.example.validation.validator.ValidationGroups.*
    
    data class UserDto(
        @field:NotNull(groups = [Update::class], message = "수정 시 ID는 필수입니다.")
        val id: Long?,
    
        @field:NotBlank(groups = [Create::class, Update::class], message = "사용자 이름은 필수입니다.")
        val username: String,
    
        @field:NotBlank(groups = [Create::class], message = "등록 시 비밀번호는 필수입니다.")
        val password: String?
    )
    ```

3. Controller에서 @Validated로 그룹 활성화

   Spring이 제공하는 @Validated 어노테이션을 사용하여 특정 그룹에 속한 규칙만 검증하도록 지정한다.

    ```kotlin
    import org.springframework.validation.annotation.Validated
    import com.example.validation.validator.ValidationGroups.*
    
    @RestController
    @RequestMapping("/api/users")
    class UserController {
    
        // 등록 API: Create 그룹만 검증
        @PostMapping
        fun createUser(@Validated(Create::class) @RequestBody userDto: UserDto): String {
            return "사용자 등록 성공"
        }
    
        // 수정 API: Update 그룹만 검증
        @PutMapping
        fun updateUser(@Validated(Update::class) @RequestBody userDto: UserDto): String {
            return "사용자 수정 성공"
        }
    }
    ```

#### 2.5. 객체 그래프 검증(Nested Validation)

DTO 내부에 또 다른 DTO 객체가 포함된 중첩 구조에서 내부 객체까지 연쇄적으로 검증해야 할 경우, 내부 객체 필드에 **`@Valid`**를 추가한다.

1. **중첩 구조 DTO 정의**

    ```kotlin
    // 주소 정보 DTO
    data class Address(
        @field:NotBlank(message = "도시는 필수입니다.")
        val city: String,
    
        @field:NotBlank(message = "상세 주소는 필수입니다.")
        val street: String
    )
    
    // 주문 요청 DTO
    data class OrderRequest(
        @field:NotNull(message = "주문 ID는 필수입니다.")
        val orderId: Long,
    
        @field:Valid // 중첩된 객체를 검증하도록 @Valid 추가
        val address: Address
    )
    ```

2. **Controller에서 사용**

    ```kotlin
    @PostMapping("/orders")
    fun createOrder(@Valid @RequestBody orderRequest: OrderRequest): String {
        // orderRequest.address 필드의 city가 비어있으면 MethodArgumentNotValidException 발생
        return "주문 성공"
    }
    ```

---
### 3. 주요 어노테이션

|어노테이션|설명|
|---|---|
|**`@NotNull`**|`null` 값을 허용하지 않는다.|
|**`@NotBlank`**|`null`, 빈 문자열(`""`), 공백만 있는 문자열(`" "`)을 허용하지 않는다. (문자열 전용)|
|**`@NotEmpty`**|`null`과 빈 값(e.g., `""`, 빈 컬렉션)을 허용하지 않는다.|
|**`@Size`**|문자열의 길이, 배열 또는 컬렉션의 크기를 검사한다. (e.g., `@Size(min=2, max=10)`)|
|**`@Min`**, **`@Max`**|숫자 값의 최소/최대치를 검사한다.|
|**`@Positive`**, **`@Negative`**|양수 또는 음수인지 검사한다.|
|**`@Email`**|올바른 이메일 형식인지 검사한다.|
|**`@Pattern`**|정규 표현식에 맞는 형식인지 검사한다.|
|**`@Future`**, **`@Past`**|날짜가 미래 또는 과거인지 검사한다.|
|**`@Valid`**|중첩된 객체의 유효성 검사를 함께 진행할 때 사용한다.|

---

### 4. 동작 원리

Spring Boot 환경에서 Bean Validation의 동작 과정은 다음과 같다.

1. **요청 접수 및 핸들러 매핑**: **DispatcherServlet**이 클라이언트의 요청을 받아 처리할 Controller(Handler)를 찾는다.

2. **Argument Resolver 실행**: **HandlerAdapter**는 Controller 메서드 실행에 필요한 파라미터를 준비하기 위해 등록된 여러 **Argument Resolver**를 사용한다. `@RequestBody`가 붙은 파라미터는 **`RequestResponseBodyMethodProcessor`**가 처리한다.

3. **객체 변환 및 검증 실행**: `RequestResponseBodyMethodProcessor`는 다음 두 가지 작업을 수행한다.

    - HTTP 요청 본문의 JSON 데이터를 DTO 객체로 변환한다.

    - 파라미터에 **`@Valid`** 어노테이션이 있는지 확인하고, 있다면 등록된 `Validator`를 호출하여 검증을 실행한다.

4. **Validator 호출**: Spring은 **`WebDataBinder`**를 통해 Jakarta Bean Validation 표준 인터페이스인 **`Validator`**를 호출한다. 특정 구현체(`Hibernate Validator`)에 직접 의존하지 않는다.

5. **검증 계획 수립**: **Hibernate Validator**는 리플렉션을 통해 DTO 클래스의 모든 필드를 스캔하여 적용된 유효성 검증 어노테이션을 확인하고 검증 계획을 세운다.

6. **ConstraintValidator 실행**: 실제 검증 로직은 `ConstraintValidator<A, T>` 인터페이스의 구현체에 정의되어 있다. Hibernate Validator는 각 어노테이션에 매핑된 `ConstraintValidator`를 찾아 `isValid()` 메서드를 호출한다.

    - 예: `@NotBlank` -> `NotBlankValidator.isValid()` 호출

7. **검증 결과 처리**:

    - **성공**: 모든 필드의 `isValid()` 메서드가 `true`를 반환하면 검증이 성공적으로 완료되고, DTO 객체는 Controller 메서드로 전달된다.

    - **실패**: 하나 이상의 필드에서 `isValid()`가 `false`를 반환하면, Hibernate Validator는 실패 정보를 담은 `ConstraintViolationException`을 발생시킨다.

8. **예외 변환**: Spring은 이 예외를 직접 처리하지 않고, Spring MVC 환경에 더 적합한 **`MethodArgumentNotValidException`**으로 변환하여 최종적으로 던진다. 이 예외는 `@RestControllerAdvice` 등을 통해 공통으로 처리될 수 있다.


---

### 5. 요약

**Bean Validation**은 어노테이션을 통해 선언적으로 데이터의 유효성을 검증하는 자바 표준 사양이다. Spring Boot는 이를 통합하여 **`@Valid`**와 **`@Validated`** 어노테이션만으로 컨트롤러단에서 DTO의 유효성을 간편하게 검증하는 기능을 제공한다. 또한, 커스텀 어노테이션과 검증 그룹 기능을 활용하여 복잡하고 상황에 맞는 유효성 검증 규칙을 체계적으로 적용할 수 있다.