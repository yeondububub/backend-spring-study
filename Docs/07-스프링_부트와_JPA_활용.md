# 실전! 스프링 부트와 JPA 활용 (1편 & 2편)

---

## Part 1. 웹 애플리케이션 개발 (활용 1편)

## 1. 프로젝트 환경설정

### 1.1 스프링 부트 3.0 이상 환경 주의사항
- **Java 17 이상** 필수 사용.
- Java EE → Jakarta EE 전환에 따라 패키지명이 `javax.*`에서 **`jakarta.*`** 로 변경됨 (`jakarta.persistence.Entity`, `jakarta.validation.Valid`, `jakarta.annotation.PostConstruct`).
- H2 데이터베이스는 **2.1.214 이상** 권장 (H2 1.4.198 이후부터 `MVCC` 옵션 제거됨).

### 1.2 JPA와 DB 설정 (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create # 실행 시점에 테이블 DROP 후 재생성 (개발 전용)
    properties:
      hibernate:
        # show_sql: true  # System.out 출력 (실무 사용 지양)
        format_sql: true

logging:
  level:
    org.hibernate.SQL: debug               # 로거를 통해 SQL 출력
    org.hibernate.orm.jdbc.bind: trace     # 스프링 부트 3.x / Hibernate 6 바인딩 파라미터 출력
```

> [!NOTE]
> **쿼리 파라미터 로그 라이브러리 (`p6spy`)**
> `com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0` (스프링 부트 3.0+)을 추가하면 바인딩된 파라미터가 포함된 완성형 SQL을 확인할 수 있다. 단, 시스템 자원을 소모하므로 운영 환경 적용 전 반드시 성능 테스트가 필요하다.

---
