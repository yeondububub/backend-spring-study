# 스프링 데이터 JPA

## 1. 프로젝트 환경설정

스프링 데이터 JPA 개발 환경 구축의 핵심은 스프링 부트 3.x 기반 프로젝트에서 요구되는 표준 자바 사양과 H2 데이터베이스 버전 호환성을 확보하고, 하이버네이트 및 SQL 파라미터 로깅 프레임워크를 최적으로 설정하는 데 있다.

### 프로젝트 생성 및 필수 선택사항

스프링 부트 스타터(start.spring.io)를 이용하여 Gradle Groovy 기반 프로젝트를 생성하며, 주요 스타터 모듈로 `spring-boot-starter-data-jpa`, `spring-boot-starter-web`, `h2`, `lombok`을 추가한다.

스프링 부트 3.0 이상 버전을 사용할 때에는 다음 필수 제약사항을 준수해야 한다:
- **자바 버전**: Java 17 이상을 반드시 사용해야 한다.
- **패키지명 변경**: 자바 및 오라클 라이선스 변경 정책에 따라 기존 `javax.*` 패키지 명칭이 `jakarta.*`로 일괄 개편되었다.
  - JPA 표준 애노테이션: `javax.persistence.Entity` -> `jakarta.persistence.Entity`
  - 라이프사이클 애노테이션: `javax.annotation.PostConstruct` -> `jakarta.annotation.PostConstruct`
  - Bean Validation 애노테이션: `javax.validation.*` -> `jakarta.validation.*`
- **H2 데이터베이스 버전**: Spring Boot 3.x 환경에 맞춰 H2 데이터베이스 2.1.214 버전 이상을 설치 및 연동해야 한다.

### 주요 라이브러리 구성 및 P6Spy 로깅 설정

- **스프링 부트 스타터 라이브러리**: `spring-boot-starter-data-jpa`는 내부적으로 AOP, JDBC, HikariCP 커넥션 풀, 하이버네이트 ORM 및 Spring Data JPA 핵심 모듈을 통합 제공한다.
- **쿼리 파라미터 로깅 (P6Spy)**: 개발 환경에서 실행되는 SQL의 바인딩 파라미터를 출력하기 위해 외부 라이브러리를 사용한다. Spring Boot 3.0 이상 환경에서는 `com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0` 이상의 버전을 의존성에 추가해야 정상 작동한다.

### 데이터베이스 연동 및 JPA 설정

`application.yml` 파일 설정을 통해 H2 데이터베이스 접속 정보와 JPA 실행 옵션을 제어한다.

```yaml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/datajpa
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        format_sql: true
logging.level:
  org.hibernate.SQL: debug
```

- `ddl-auto: create`: 애플리케이션 시작 시점에 기존 스키마를 drop하고 엔티티 매핑 정보를 기반으로 테이블을 재생성한다.
- `logging.level.org.hibernate.SQL: debug`: 하이버네이트가 생성하는 SQL을 로거를 통해 남긴다.
- **IntelliJ 실행 속도 최적화**: IntelliJ IDEA 환경에서 Gradle을 통한 애플리케이션 실행 방식을 `IntelliJ IDEA` 자바 직접 실행 옵션(Preferences -> Build Tools -> Gradle -> Build and run using: IntelliJ IDEA)으로 변경하면 실행 속도를 대폭 향상시킬 수 있다.