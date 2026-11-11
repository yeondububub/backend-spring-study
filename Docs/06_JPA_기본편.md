# JPA 기본편

## 1. JPA 소개 및 ORM의 필요성

```
+-------------------------------------------------------------------------+
|                         객체지향 vs 관계형 DB 패러다임                        |
+------------------------------------+------------------------------------+
|               객체 (Java)           |          관계형 DB (RDB)             |
+------------------------------------+------------------------------------+
|  - 참조를 통한 연관관계 탐색             |  - 외래 키(FK)와 JOIN을 통한 조회        |
|  - 상속 구조 (다형성)                  |  - 슈퍼타입 서브타입 모델                |
|  - 메모리 주소값 동일성 (==)            |  - PK(기본키) 기준 행 구분              |
+------------------------------------+------------------------------------+
```

### 1.1 SQL 중심 개발의 문제점

- **지루한 CRUD 작성의 반복**: 테이블마다 무수한 `INSERT`, `UPDATE`, `SELECT`, `DELETE` SQL을 수작업으로 작성해야 함.
- **객체와 RDB 간의 패러다임 불일치**: 객체지향은 **추상화, 캡슐화, 정보의닉, 상속, 다형성**을 지향하지만, RDB는 **데이터의 정규화된 저장 및 집합 연산**을 지향함.

### 1.2 패러다임 불일치의 4대 요소

1. **상속**: 객체는 상속 관계가 있지만 RDB는 상속 개념이 없음 (슈퍼타입-서브타입 관계 모델링만 존재).
2. **연관관계**: 객체는 **참조(`member.getTeam()`)**를 사용하여 한 방향으로 탐색하지만, DB는 **외래 키(`JOIN ON ...`)**를 이용해 양방향 조인이 가능함.
3. **객체 그래프 탐색**: 객체는 자유롭게 그래프를 탐색해야 하지만, SQL 중심 개발에서는 처음에 실행한 SQL에 의해 탐색 범위가 결정되어 엔티티를 신뢰할 수 없음.
4. **비교**: RDB는 PK 값으로 행을 구분하지만, 자바는 참조값(`==`)과 값(`equals()`) 비교 방식을 가짐.

### 1.3 ORM과 JPA의 개념

- **ORM (Object-Relational Mapping)**: 객체와 관계형 데이터베이스의 데이터를 자동으로 매핑해주는 기술.
- **JPA (Java Persistence API)**: 자바 진영의 **ORM 기술 표준 API 명세**. 인터페이스 모음이며 대표적인 구현체로 **Hibernate(하이버네이트)**가 있음.

```
+-----------------------------------------------------------------------------+
|                              JPA 아키텍처                                     |
+-----------------------------------------------------------------------------+
|   Application  --->  JPA (표준 API)  --->  Hibernate (구현체)  --->  JDBC/DB   |  
+-----------------------------------------------------------------------------+
```

> [!NOTE]  
> **JPA를 사용해야 하는 이유**
> - **생산성**: `em.persist(member)`, `em.find(Member.class, id)` 등의 간단한 메서드로 CRUD 해결.
> - **유지보수**: 필드 추가/수정 시 SQL을 전부 수정할 필요 없이 엔티티 클래스만 수정.
> - **패러다임 불일치 해결**: 상속, 연관관계, 객체 그래프 탐색, 동일성 보장을 JPA가 처리.
> - **성능 최적화**: 1차 캐시와 동일성 보장, 쓰기 지연(Transactional Write-behind), 지연 로딩(Lazy Loading).

## 2. JPA 시작 및 기초 애플리케이션 개발

### 2.1 JPA 설정 (persistence.xml)

JPA는 `META-INF/persistence.xml` 설정을 통해 데이터베이스 연결 정보와 방언(Dialect)을 설정함.

```xml
<persistence version="2.2">
    <persistence-unit name="hello">
        <properties>
            <!-- 필수 속성 -->
            <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="javax.persistence.jdbc.user" value="sa"/>
            <property name="javax.persistence.jdbc.password" value=""/>
            <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>

            <!-- 옵션 -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="create"/>
        </properties>
    </persistence-unit>
</persistence>
```

### 2.2 EntityManagerFactory와 EntityManager

- **EntityManagerFactory**: 애플리케이션 생성 시점에 단 하나만 생성하여 전체에서 공유. (비용이 매우 큼)
- **EntityManager**: 고객의 요청이 올 때마다 생성하고 사용 후 반드시 닫음. **쓰레드 간 절대 공유 금지!**

```java
public class JpaMain {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();
        EntityTransaction tx = em.getTransaction();

        tx.begin(); // 트랜잭션 시작

        try {
            Member member = new Member();
            member.setId(1L);
            member.setName("HelloA");

            em.persist(member); // 영속화

            tx.commit(); // 트랜잭션 커밋
        } catch (Exception e) {
            tx.rollback();
        } finally {
            em.close(); // 엔티티 매니저 종료
        }

        emf.close();
    }
}
```

> [!IMPORTANT] 
> JPA의 모든 데이터 변경은 트랜잭션 안에서 실행되어야 한다.


## 3. 영속성 관리 (Persistence Management)

### 3.1 영속성 컨텍스트 (Persistence Context)

- **"엔티티를 영구 저장하는 환경"**을 의미하는 논리적 개념.
- `EntityManager`를 통해 영속성 컨텍스트에 접근하고 관리함.

### 3.2 엔티티의 생명주기 (Entity Lifecycle)

```
+-------------------------------------------------------------------------+
|                           엔티티 생명주기 상태                               |
+-----------------+-------------------------------------------------------+
| 비영속 (New)      | 영속성 컨텍스트와 전혀 관계가 없는 새로운 상태                   |
| 영속 (Managed)   | 영속성 컨텍스트에 의해 관리되는 상태 (em.persist)              |
| 준영속 (Detached) | 영속성 컨텍스트에 저장되었다가 분리된 상태 (em.detach)          |
| 삭제 (Removed)   | 삭제된 상태 (em.remove)                                  |
+-----------------+-------------------------------------------------------+
```

```java
// 1. 비영속 상태 (New)
Member member = new Member();
member.setId(100L);
member.setName("MemberA");

// 2. 영속 상태 (Managed) - 영속성 컨텍스트에 저장됨 (DB 저장 X)
em.persist(member);

// 3. 준영속 상태 (Detached) - 영속성 컨텍스트에서 분리
em.detach(member);

// 4. 삭제 상태 (Removed) - DB 레코드 삭제 요청
em.remove(member);
```

### 3.3 영속성 컨텍스트의 핵심 이점

1. **1차 캐시 및 동일성(Identity) 보장**:
   - 메모리 내 1차 캐시에서 먼저 조회하여 DB 접근 횟수 감소.
   - 같은 트랜잭션 내에서는 `a == b` (동일성 비교)가 `true`를 보장.

2. **트랜잭션을 지원하는 쓰기 지연 (Transactional write-behind)**:
   - `em.persist()` 호출 시 DB에 바로 SQL을 보내지 않고, **쓰기 지연 SQL 저장소**에 모아두었다가 트랜잭션 커밋 시점에 한 번에 전송 (`flush`).

3. **변경 감지 (Dirty Checking)**:
   - 1차 캐시에 엔티티를 저장할 때 최초 상태인 **스냅샷**을 함께 저장.
   - 트랜잭션 커밋 시점에 엔티티와 스냅샷을 비교하여 변경 사항이 있으면 `UPDATE` SQL을 자동으로 생성 및 등록.

```java
Member member = em.find(Member.class, 100L);
member.setName("UpdatedName"); // em.update()나 em.persist()를 호출할 필요 없음!
// tx.commit() 시점에 변경 감지가 작동하여 UPDATE SQL 자동 실행
```

### 3.4 플러시 (Flush)

영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화하는 작업.
- **플러시 발생시점**:
  1. `em.flush()` 직접 호출
  2. 트랜잭션 커밋 시 자동 호출
  3. JPQL 쿼리 실행 시 자동 호출

## 4. 엔티티 매핑 (Entity Mapping)

### 4.1 객체-테이블 매핑 (`@Entity`, `@Table`)
- `@Entity`: JPA가 관리할 클래스임을 지정. 기본 생성자(Public/Protected) 필수!
- `@Table`: 엔티티와 매핑할 DB 테이블 지정 (생략 시 엔티티 이름 사용).

### 4.2 DDL 자동 생성 옵션 (`hibernate.hbm2ddl.auto`)
- `create`: 기존 테이블 삭제 후 다시 생성 (DROP + CREATE)
- `create-drop`: create와 같으나 애플리케이션 종료 시 DROP
- `update`: 변경분만 반영 (컬럼 추가만 가능, 삭제 불가능)
- `validate`: 엔티티와 테이블이 정상 매핑되었는지만 검증
- `none`: 사용하지 않음

> [!WARNING]  
> **운영 환경에서는 절대로 create, create-drop, update를 사용하면 안 됩니다!** (validate 또는 none 권장)

### 4.3 필드와 컬럼 매핑 애노테이션

- `@Column`: 컬럼명, `nullable`, `unique`, `length` 지정.
- `@Enumerated`: 자바 enum 타입 매핑. **반드시 `EnumType.STRING` 사용!** (`ORDINAL`은 순서 숫자로 저장되어 버그 유발).
- `@Temporal`: 날짜 타입 매핑 (`TemporalType.DATE`, `TIME`, `TIMESTAMP`). 자바 8+ `LocalDate`, `LocalDateTime` 사용 시 생략 가능.
- `@Lob`: 대용량 데이터 (문자 CLOB, 바이너리 BLOB).
- `@Transient`: DB 매핑에서 제외 (메모리 임시 변수).

```java
@Entity
@Table(name = "MEMBERS")
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false, length = 100)
    private String username;

    @Enumerated(EnumType.STRING) // 중요!
    private RoleType roleType;

    private LocalDateTime createdDate;

    @Lob
    private String description;

    @Transient
    private int tempCalculatedValue;
}
```

### 4.4 기본 키 매핑 전략

1. **IDENTITY**: 기본 키 생성을 DB에 위임 (MySQL `AUTO_INCREMENT`). `em.persist()` 시점에 즉시 `INSERT` SQL 실행하여 PK 조회.
2. **SEQUENCE**: DB 시퀀스 오브젝트 사용 (Oracle, H2). `@SequenceGenerator`로 `allocationSize` 설정하여 성능 최적화.
3. **TABLE**: 키 생성 전용 테이블 사용 (모든 DB 적용 가능하나 성능 불리).
4. **AUTO**: 방언에 따라 위 3가지 중 자동 선택.

## 5. 연관관계 매핑 기초

### 5.1 연관관계가 필요한 이유

테이블은 **외래 키(FK)**로 조인을 하고, 객체는 **참조**를 사용함. 외래 키 필드를 객체에 그대로 저장하면 객체지향적인 그래프 탐색이 불가능해짐.

### 5.2 단방향 연관관계 (`@ManyToOne`, `@JoinColumn`)

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String username;

    @ManyToOne
    @JoinColumn(name = "TEAM_ID") // FK 컬럼 매핑
    private Team team;
}
```

### 5.3 양방향 연관관계와 연관관계의 주인 (Owner)

```
+-------------------------------------------------------------------------+
|                         양방향 연관관계 구조                                 |
+------------------------------------+------------------------------------+
|  Member (외래키 소유 / 주인)           |  Team (가짜 매핑 / 읽기 전용)          |
+------------------------------------+------------------------------------+
|  @ManyToOne                        |  @OneToMany(mappedBy = "team")     |
|  @JoinColumn(name = "TEAM_ID")     |  private List<Member> members;     |
|  private Team team;                |                                    |
+------------------------------------+------------------------------------+
```

#### 양방향 매핑 규칙:

1. 객체의 두 관계 중 하나를 **연관관계의 주인**으로 지정해야 함.
2. **외래 키(FK)가 위치한 곳을 주인으로 정함!** (예: `Member.team`)
3. **주인만이 외래 키를 등록 및 수정**할 수 있음.
4. **주인이 아닌 쪽은 `mappedBy` 속성으로 주인을 지정**하며, 읽기만 가능함.

### 5.4 양방향 매핑 시 주의사항 & 편의 메서드

- 순수 객체 상태를 고려하여 **양쪽 모두에 값을 세팅**해야 함.
- **연관관계 편의 메서드**를 작성하여 실수를 방지함.

```java
// Member 엔티티 내부의 연관관계 편의 메서드
public void changeTeam(Team team) {
    this.team = team;
    team.getMembers().add(this); // 양방향 연관관계 동시 설정
}
```

> [!IMPORTANT] 
> 양방향 매핑 시 `toString()`, Lombok, JSON 변환 라이브러리에서 무한 루프를 조심해야 한다.

## 6. 다양한 연관관계 매핑

### 6.1 연관관계 고려사항 3가지

1. **다중성**: N:1, 1:N, 1:1, N:M
2. **방향**: 단방향, 양방향
3. **연관관계의 주인**: 외래 키를 관리하는 주체

### 6.2 주요 연관관계 유형 비교

| 연관관계 유형 | 특이사항 및 권장 여부 |
| :--- | :--- |
| **다대일 (N:1)** | 가장 많이 사용하는 연관관계. 항상 N쪽에 FK가 존재하므로 N쪽이 주인! |
| **일대다 (1:N)** | 1쪽이 연관관계의 주인. FK가 반대편 테이블에 있어 **추가 UPDATE SQL 발생**. **다대일 양방향 매핑 권장!** |
| **일대일 (1:1)** | 주 테이블 또는 대상 테이블에 FK 선택 가능. FK에 UNIQUE 제약조건 추가 필요. |
| **다대다 (N:M)** | `@ManyToMany`는 실무 사용 금지! **연결 엔티티를 추가하여 1:N, N:1로 승격**해야 함. |

### 6.3 다대다(N:M)의 한계와 실무 대안

```
[Member] 1 --- * [OrderItem (연결 엔티티)] * --- 1 [Item]
```

`@ManyToMany`는 중간 테이블에 추가 컬럼(주문 수량, 주문 일시 등)을 넣을 수 없고 은닉된 쿼리가 나가므로, **중간 연결 엔티티(`OrderItem`)를 만들어 `@OneToMany`, `@ManyToOne`으로 매핑**함.

## 7. 고급 매핑 (Inheritance & MappedSuperclass)

### 7.1 상속관계 매핑 전략
관계형 데이터베이스의 **슈퍼타입 서브타입 관계** 모델링 기법을 객체 상속과 매핑함.

**상속관계 매핑 전략 비교**

| 전략 | 장점 | 단점 |
| :--- | :--- | :--- |
| **조인 전략**<br>(`JOINED`) | 정규화, 저장공간 효율, FK 무결성<br>*(가장 정석적인 권장 전략)* | 조인 쿼리, INSERT 2번 |
| **단일 테이블**<br>(`SINGLE_TABLE`) | 조인이 없어 조회 성능 빠름<br>*(기본값)* | 자식 컬럼 NULL 허용,<br>테이블이 커질 수 있음 |
| **구현 클래스**<br>(`TABLE_PER_CLASS`) | 서브타입 구분 명확 | UNION 쿼리 발생,<br>**실무 비추천!** |


```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED) // 조인 전략 선택
@DiscriminatorColumn(name = "DTYPE") // 구분 컬럼
public abstract class Item {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private int price;
}

@Entity
@DiscriminatorValue("B")
public class Book extends Item {
    private String author;
    private String isbn;
}
```

### 7.2 `@MappedSuperclass`

- 공통 매핑 정보(예: `createdDate`, `lastModifiedDate`)가 필요할 때 부모 클래스에 선언.
- **엔티티가 아니며 테이블과 매핑되지 않음.** 단순히 매핑 정보만 자식에게 제공.
- 직접 생성할 일이 없으므로 **추상 클래스(abstract class)**로 선언 권장.

```java
@MappedSuperclass
public abstract class BaseEntity {
    private String createdBy;
    private LocalDateTime createdDate;
    private String lastModifiedBy;
    private LocalDateTime lastModifiedDate;
}
```
