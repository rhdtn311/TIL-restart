이 문서는 사용자의 초안을 바탕으로 Gemini가 체계적으로 구조화하고 내용을 다듬어 작성했습니다.

### 1\. 스프링 부트 개요

스프링 부트(**Spring Boot**)는 스프링 프레임워크를 기반으로 개발 편의성을 극대화하기 위해 설계된 프레임워크다. 복잡한 XML 또는 자바 설정 파일을 최소화하고, 개발자가 **비즈니스 로직**에 집중할 수 있도록 돕는 것을 목표로 한다.

#### 1.1. 주요 특징

* **자동 구성(Auto-Configuration)**: 클래스패스와 환경 설정을 기반으로 애플리케이션에 필요한 빈(Bean)들을 자동으로 등록한다.
* **내장 서버**: 톰캣(Tomcat), 제티(Jetty), 언더토우(Undertow)와 같은 웹 서버를 내장하고 있어 별도의 서버 설치 및 설정 없이 실행할 수 있다.
* **스타터(Starter) 의존성**: 공통적으로 사용되는 라이브러리들을 묶어 제공하는 의존성 집합으로, 관련 자동 구성 설정이 함께 포함되어 있다.
* **간단한 외부 설정**: `application.yml` 또는 `application.properties` 파일을 통해 간편하게 외부 설정을 관리할 수 있다.

-----

### 2\. 스프링 부트 실행 및 자동 구성 원리

#### 2.1. 애플리케이션 실행 흐름

스프링 부트 애플리케이션은 `main()` 메서드 내 `SpringApplication.run()` 호출을 시작으로 동작한다. 이 과정에서 핵심적으로 `@SpringBootApplication` 애노테이션을 해석한다.

* `@SpringBootApplication`: 다음 세 가지 주요 애노테이션을 포함하는 복합 애노테이션이다.
    * `@EnableAutoConfiguration`: 자동 구성을 활성화한다.
    * `@ComponentScan`: `@Component`, `@Service`, `@Repository` 등이 붙은 빈들을 자동으로 스캔하여 IoC 컨테이너에 등록한다.
    * `@SpringBootConfiguration`: 해당 클래스를 스프링 부트의 설정 클래스로 지정한다.

#### 2.2. 자동 구성 과정

1.  **후보군 로딩**: `@EnableAutoConfiguration`이 활성화되면 `AutoConfigurationImportSelector`가 동작하여 **자동 구성 후보군**을 불러온다.
2.  **조건 검증**: 로드된 각 `*AutoConfiguration` 클래스에 정의된 \*\*조건부 애노테이션(`@Conditional`)\*\*을 검사한다.
3.  **빈 등록**: 모든 조건을 만족하는 자동 구성 클래스에 대해 관련 빈들을 IoC 컨테이너에 등록한다. 조건에 맞지 않으면 해당 빈 등록은 건너뛴다.
4.  **애플리케이션 구동**: 모든 빈 등록이 완료되면 IoC 컨테이너가 초기화되고, 내장 서버가 실행되어 애플리케이션이 구동된다.

-----

### 3\. 자동 구성 상세 설명

스프링 부트의 자동 구성 기능은 `spring-boot-autoconfigure` 모듈에 포함된 수많은 `*AutoConfiguration` 클래스를 통해 구현된다. 개발자가 `starter` 의존성을 추가하면, 해당 의존성과 관련된 자동 구성 클래스가 후보군에 포함된다.

#### 3.1. 주요 조건부 애노테이션

* `@ConditionalOnClass`: 특정 클래스가 클래스패스에 존재할 때만 적용된다.
* `@ConditionalOnMissingBean`: 동일한 타입의 빈이 IoC 컨테이너에 이미 등록되어 있지 않을 때만 적용된다. 이를 통해 개발자가 직접 등록한 빈이 우선적으로 사용되도록 한다.
* `@ConditionalOnProperty`: `application.yml` 파일에 특정 속성(property)이 설정되어 있을 때만 적용된다.

-----

### 4\. 데이터베이스 자동 구성 예시 (`DataSourceAutoConfiguration`)

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
                .url(props.getUrl())
                .username(props.getUsername())
                .password(props.getPassword())
                .build();
    }
}
```

위 코드는 스프링 부트가 데이터베이스 연결을 자동으로 구성하는 과정을 보여준다.

1.  **`@ConditionalOnClass(DataSource.class)`**: 애플리케이션의 클래스패스에 `DataSource.class`가 있는지 확인한다. 이는 JDBC 드라이버가 존재함을 의미한다.
2.  **`@ConditionalOnMissingBean`**: IoC 컨테이너에 이미 `DataSource` 타입의 빈이 없는지 확인한다.
3.  두 조건을 모두 만족할 경우, `application.yml`의 `spring.datasource.*` 속성값을 읽어 **`DataSource` 빈**을 자동으로 생성하고 등록한다.

결과적으로, 개발자는 단순히 `application.yml`에 데이터베이스 연결 정보만 명시하면 별도의 설정 없이 `DataSource`를 사용할 수 있다.

-----

### 5\. 자동 구성 후보군 탐색 방법

스프링 부트는 특정 파일을 읽어 자동 구성 후보군을 찾는다.

* **스프링 부트 2.x**: `META-INF/spring.factories` 파일에서 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 속성값을 읽어온다.

<!-- end list -->

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

* **스프링 부트 3.x**: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일에서 목록을 읽어온다.

<!-- end list -->

```
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

-----

### 6\. 개발자가 얻는 이점

* **설정보다 관례(Convention over Configuration)**: 복잡한 설정보다는 약속된 규칙을 따름으로써 생산성을 높인다.
* **유연성**: 자동 구성된 빈은 언제든지 개발자가 직접 등록한 빈으로 **덮어쓸 수 있다**.
* **가시성**: `--debug` 옵션을 사용하면 어떤 자동 구성이 적용되었고, 어떤 이유로 제외되었는지 자세히 확인할 수 있다.

-----

### 요약

스프링 부트는 **자동 구성**을 핵심 기능으로 삼아 개발자가 복잡한 환경 설정 대신 **비즈니스 로직**에 집중할 수 있도록 돕는 프레임워크다. `@EnableAutoConfiguration`은 클래스패스와 환경 설정을 기반으로 필요한 빈들을 자동으로 등록하며, 이는 `META-INF` 디렉터리의 특정 파일을 통해 후보군을 탐색하고 `@Conditional` 애노테이션으로 적용 여부를 결정하는 방식으로 동작한다. 이러한 자동화 덕분에 개발자는 `application.yml` 파일과 같은 간단한 외부 설정만으로도 애플리케이션을 손쉽게 구성할 수 있다.