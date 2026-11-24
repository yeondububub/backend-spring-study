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

## 2. 예제 도메인 모델

데이터 접근 기술을 검증하기 위한 도메인 모델은 회원(`Member`)과 팀(`Team`) 간의 다대일(N:1) 양방향 연관관계를 중심으로 설계한다.

### 엔티티 구조 및 양방향 매핑 설계

- **Member 엔티티**: 식별자(`member_id`), 회원명(`username`), 나이(`age`), 팀 참조(`team`) 필드로 구성된다. 지연 로딩을 위해 `@ManyToOne(fetch = FetchType.LAZY)`을 지정하고 `@JoinColumn(name = "team_id")`을 선언하여 데이터베이스 외래키(FK)의 등록 및 수정 권한을 갖는 연관관계의 주인이 된다.
- **Team 엔티티**: 식별자(`team_id`), 팀명(`name`), 회원 목록(`members`) 필드로 구성된다. `@OneToMany(mappedBy = "team")`을 부착하여 `Member.team` 연관관계 주인에 대응하는 읽기 전용 참조 관계를 구성한다.

### 롬복(Lombok) 활용 및 엔티티 구현 수칙

단순한 코드 절감을 넘어 JPA 동작 메커니즘을 고려한 엔티티 구현 규칙은 다음과 같다.

| 구 분 | 사용 애노테이션 및 설정 | 실무 설계 수칙 및 사유 |
| :--- | :--- | :--- |
| **Setter 관리** | `@Setter` 사용 금지 또는 최소화 | 엔티티 인스턴스의 무분별한 상태 변경을 방지하고 비즈니스 메서드로 변경 지점을 명확히 제어한다. |
| **기본 생성자** | `@NoArgsConstructor(access = AccessLevel.PROTECTED)` | JPA 스펙 표준상 기본 생성자는 필수이나, 객체의 외부 직접 생성을 차단하기 위해 생성자 접근 수준을 PROTECTED로 제한한다. |
| **ToString 설정** | `@ToString(of = {"id", "username", "age"})` | 양방향 연관관계가 설정된 엔티티 필드를 ToString 대상에 포함할 경우 상호 호출에 의한 순환 참조(무한 루프)가 발생하므로 자체 필드만 지정한다. |
| **연관관계 메서드**| `changeTeam(Team team)` | 객체 상태와 연관관계 편의성을 위해 양쪽 객체의 참조를 한 번에 안전하게 설정하는 편의 메서드를 작성한다. |

## 3. 공통 인터페이스 기능

스프링 데이터 JPA는 상투적으로 반복되는 기본 CRUD 코드를 제거하고, 인터페이스 선언만으로 데이터 접근 계층(DAO)을 자동으로 완성하는 공통 인터페이스 기능을 제공한다.

### 순수 JPA 리포지토리와의 구조적 비교

- **순수 JPA 기반 리포지토리 (`MemberJpaRepository`)**: `@PersistenceContext`를 활용해 `EntityManager`를 주입받은 후 `em.persist()`, `em.remove()`, `em.find()`, 그리고 JPQL 구문을 통해 `findAll()` 및 `count()` 메서드를 직접 개발자가 작성해야 했다.
- **스프링 데이터 JPA 리포지토리 (`MemberRepository`)**: `public interface MemberRepository extends JpaRepository<Member, Long>`과 같이 인터페이스를 선언하면, 스프링 데이터 JPA가 애플리케이션 실행 시점에 동적 프록시 기술로 구현 클래스를 자동 생성하여 스프링 빈으로 등록한다 (`@Repository` 애노테이션 생략 가능).

### 공통 인터페이스 계층 구조

스프링 데이터 공통 기술 패키지에서 출발하여 JPA 특화 기능으로 확장되는 계층적 인터페이스 구조를 갖는다.

- `org.springframework.data.repository.Repository` (최상위 마커 인터페이스)
  - `CrudRepository` (기본 생성, 조회, 수정, 삭제 기능 제공)
    - `PagingAndSortingRepository` (정렬 및 페이징 기능 추가)
      - `org.springframework.data.jpa.repository.JpaRepository` (JPA 특화 메서드 제공: `findAll()`, `saveAndFlush()`, `deleteInBatch()` 등)

### 주요 제공 메서드 동작 특성

- `save(S)`: 새로운 엔티티인 경우 저장(`persist`)하고, 이미 존재하는 엔티티인 경우 병합(`merge`)을 실행한다.
- `delete(T)`: 엔티티 단건을 삭제하며 내부적으로 `EntityManager.remove()`를 수행한다.
- `findById(ID)`: 식별자를 기반으로 엔티티를 조회하고 `Optional<T>`으로 반환한다 (내부적으로 `EntityManager.find()` 호출).
- `getOne(ID)`: 엔티티를 프록시 객체 상태로 조회한다 (내부적으로 `EntityManager.getReference()` 호출).
- `findAll(...)`: 조건 없이 모든 엔티티를 조회하거나, `Sort` 및 `Pageable` 파라미터를 적용하여 정렬 및 페이징 조회를 수행한다.
