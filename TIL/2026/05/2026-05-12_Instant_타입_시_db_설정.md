# Instant 타입 시 db 설정

> 날짜: 2026-05-12
> 원본 노션: [링크](https://www.notion.so/Instant-db-35eedfd5a4768075addde414f23042a7)

---

Instant 타입 인 경우에 db 도 뭐 따로 설정해야 하지 않냐?

## 1. DB 컬럼 타입 설정 (TIMESTAMP vs DATETIME)

대부분의 현대적인 관계형 DB에서는 Instant를 TIMESTAMP 타입으로 매핑하는 것이 가장 권장됩니다.

- 추천: TIMESTAMP (시간대 정보 포함 가능)
- 비추천: DATETIME (시간대 정보가 없어 DB 설정에 따라 달라질 수 있음) [2]
SQL 예시 (MySQL/PostgreSQL):

- - 시간대 정보를 저장하는 것이 가장 안전함 (UTC)CREATE TABLE my_table ( created_at TIMESTAMP(6) NOT NULL - 6은 마이크로초 정밀도
);
## 2. 하이버네이트(Hibernate) 설정 (버전 주의)

Hibernate 6 버전부터 Instant 매핑 방식이 변경되었습니다. [3]

- Hibernate 6 이상: Instant를 기본적으로 TIMESTAMP_UTC로 매핑하여 DB의 시간대와 상관없이 UTC로 저장합니다.
- Hibernate 5: Instant를 매핑할 때 해당 서버의 JVM 로컬 타임존으로 변환하여 저장할 수도 있으므로 주의해야 합니다. UTC로 저장되도록 드라이버 설정을 확인해야 합니다. [2, 4, 5]
하이버네이트 설정 (properties/yml):

# 하이버네이트 6에서 Instant를 TIMESTAMP로 명시적 매핑
spring.jpa.properties.hibernate.type.preferred_instant_jdbc_type=TIMESTAMP

## 3. JDBC 드라이버 타임존 설정 (가장 중요)

Java 애플리케이션과 DB 간의 데이터 전송 시 타임존 혼선을 막기 위해 UTC로 설정해야 합니다.

- JDBC 연결 URL에 추가: ?serverTimezone=UTC (MySQL 예시)
- 애플리케이션 시작 시 설정: TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
## 요약

Instant 타입을 쓸 때는 DB, 애플리케이션, JPA 모두 UTC를 기준으로 동작하도록 설정하는 것이 데이터 불일치를 막는 가장 좋은 방법입니다. [5, 6]

[1] [https://docs.oracle.com](https://docs.oracle.com/javase/8/docs/api/java/time/OffsetDateTime.html)

[2] [https://thorben-janssen.com](https://thorben-janssen.com/hibernate-jpa-date-and-time/)

[3] [https://stackoverflow.com](https://stackoverflow.com/questions/75733887/java-time-instant-field-cannot-be-filled-by-jpql-query-using-current-timestamp)

[4] [https://github.com](https://github.com/jhipster/generator-jhipster/issues/24437)

[5] [https://stackoverflow.com](https://stackoverflow.com/questions/70522389/jpa-instant-zoneddatetime-database-mapping)

[6] [https://stackoverflow.com](https://stackoverflow.com/questions/75300125/how-to-map-an-instant-field-as-timestamp-without-timezone-in-hibernate-6)







### Soft Delete 정책



### EmbeddedId equals hashcode



### 좋아요 멱등성

```java
public void like(final OffsetDateTime now) {
    if (likedAt != null) {
        return;
    }
    likedAt = now;
}

public void unlike() {
    if (likedAt == null) {
        return;
    }
    likedAt = null;
}
```

앞서 보신 엔티티 내부의 if (likedAt != null) 처리는 도메인 객체 레벨에서의 논리적 멱등성입니다. 하지만 실제 백엔드 시스템에서는 여러 스레드나 분산 서버에서 동시에 요청이 쏟아질 때 이를 완벽하게 방어해야 하므로, 인프라 및 아키텍처 레벨에서 멱등성을 보장하는 방식들을 주로 사용합니다.

실무에서 "좋아요"와 같은 기능의 멱등성을 보장하기 위해 사용하는 대표적인 아키텍처 패턴들은 다음과 같습니다.

---

### 1. 데이터베이스 레벨의 멱등성 (Storage Idempotency)

가장 확실하고 데이터 정합성이 강력하게 보장되는 방식입니다.

- Unique Index (유니크 제약조건) 활용:
- UPSERT (Insert or Update) 쿼리 활용:
### 2. Redis를 활용한 단일 원자적 처리 (Infrastructural Idempotency)

트래픽이 매우 많아 DB까지 요청이 가는 것 자체를 막아야 하거나, 짧은 시간 내의 중복 요청(따닥!)을 앞단에서 쳐내고 싶을 때 강력한 방식입니다.

- Redis SET (NX 옵션):Bash
### 3. API API Idempotency Key (멱등성 키 패턴)

결제(Payment) 도메인이나 주문 시스템처럼 "절대로 두 번 처리되면 안 되는" 크리티컬한 로직에서 주로 사용하는 글로벌 표준 패턴입니다. "좋아요"에 쓰기엔 오버엔지니어링일 수 있지만, 개념적으로 가장 완벽한 형태입니다.

1. 클라이언트(프론트엔드/앱)가 "좋아요" 요청을 보낼 때, 고유한 UUID(예: Idempotency-Key: 123e4567-...)를 생성하여 HTTP 헤더에 담아 보냅니다.
1. 서버는 이 키를 Redis나 DB에 저장된 멱등성 키 저장소에서 조회합니다.
1. 처음 보는 키라면: 비즈니스 로직(DB 저장 등)을 정상 수행하고, 그 결과(HTTP 200 OK 등)를 해당 키와 함께 맵핑하여 캐싱합니다.
1. 이미 존재하는 키라면: 저장된 키에 맵핑된 '이전 응답 결과'를 꺼내서 비즈니스 로직 수행 없이 그대로 반환합니다.
---

### 💡 실무 적용 요약

보통 "좋아요" 기능의 경우 DB UPSERT 쿼리로 처리하거나, 조회/쓰기 트래픽이 몰리는 아키텍처라면 Redis SET NX를 앞단에 두어 방어하는 방식을 가장 많이 채택합니다.

/////

### equals 사용 이유

"그냥 비교(== 연산자)"를 쓰면 로컬 테스트에서는 잘 되다가 실제 서비스(운영 환경)에서 갑자기 버그가 터지는 대참사가 발생할 수 있기 때문입니다.

그 이유를 명확히 이해하려면 자바의 객체 비교 방식과 JPA 엔티티의 특징을 알아야 합니다.

### 1. getId()의 반환 타입은 객체(Long)입니다.

JPA 엔티티를 설계할 때 ID 필드는 보통 원시 타입(long)이 아니라 래퍼 클래스인 Long 객체를 사용합니다. (DB에 저장되기 전에는 ID가 null이어야 하기 때문입니다.)

자바에서 객체를 ==로 비교하면 '실제 데이터 값'이 아니라 '메모리 주소(참조값)'를 비교하게 됩니다.

- == : 두 객체가 메모리상 완전히 동일한 위치(주소)에 있는가?
- equals() : 두 객체의 실제 데이터 값이 같은가?
### 2. 최악의 함정: "어? 테스트할 땐 == 써도 잘 되던데요?" (JVM 캐싱)

자바(JVM)는 메모리를 아끼기 위해 -128 부터 127 까지의 Long 및 Integer 값은 미리 캐싱(저장)해 두고 재사용합니다.

이것이 아주 무서운 함정을 만듭니다.

Java

# 

// ID가 100일 때 (127 이하) -> 캐시된 동일한 객체를 반환함
Long id1 = 100L;
Long id2 = 100L;
System.out.println(id1 == id2);      // true (테스트 성공!)
System.out.println(id1.equals(id2)); // true

// ID가 1000일 때 (127 초과) -> 캐시 범위를 벗어나서 각각 새로운 객체를 생성함
Long id3 = 1000L;
Long id4 = 1000L;
System.out.println(id3 == id4);      // false (운영 환경에서 갑자기 권한 에러 발생!!)
System.out.println(id3.equals(id4)); // true (안전함)

만약 ==로 작성했다면, 서비스 초기에는 회원가입 수가 적어서 ID가 127 이하일 테니 완벽하게 작동합니다. 그러다 가입자가 128명을 넘어서는 순간부터 비교 로직이 전부 false를 반환하며 원인을 찾기 힘든 심각한 버그(예: 본인 글인데 삭제가 안 됨)가 터지게 됩니다.

---

### 💡 요약 및 권장 사항

결론적으로, getId()로 가져온 값은 Long 타입의 객체이므로, 값의 동등성을 비교하려면 반드시 .equals()를 사용해야 합니다.

더 안전하게 작성하는 팁:

만약 repostRequestPost.getUser().getId()가 null일 가능성이 아주 조금이라도 있다면 NullPointerException이 발생할 수 있습니다. 이를 방지하기 위해 자바의 Objects.equals()를 사용하는 것이 실무에서 더 안전하고 깔끔한 패턴입니다.

Java

# 

```java
// 추천하는 안전한 비교 방식
if (Objects.equals(repostRequestPost.getUser().getId(), user.getId())) {
    // 로직 수행
}
```

///////

### 실무에서 대리키 복합키(좋아요 , 게시물 등)

실무에서 대리키(Surrogate Key, 예: Auto Increment ID)를 쓸지, 자연키/복합키(Natural/Composite Key)를 쓸지는 백엔드 개발자와 DBA 사이에서 항상 나오는 단골 토론 주제입니다.

결론부터 말씀드리면, '좋아요(Like)'와 같은 단순 다대다(N:M) 매핑 테이블에서는 대리키(단일 ID)를 추가하는 것보다 복합키를 PK로 쓰는 것이 오히려 성능과 정합성 면에서 더 유리한 경우가 많습니다.

성능 저하가 올 것이라는 걱정은 절반은 맞고 절반은 틀립니다. 그 이유를 데이터베이스 내부의 인덱스 동작 방식과 함께 설명해 드리겠습니다.

### 1. 진짜 성능 이슈: 대리키(ID)를 쓰면 인덱스를 '두 번' 관리해야 합니다.

만약 Like 테이블에 id라는 단일 PK를 둔다고 가정해 보겠습니다.

SQL

- - 대리키를 사용한 설계CREATE TABLE likes ( id BIGINT AUTO_INCREMENT PRIMARY KEY, user_id BIGINT, post_id BIGINT
);
이 상태에서 동시성 이슈나 중복 좋아요를 막으려면 어떻게 해야 할까요? 앞서 말씀드린 대로 (user_id, post_id)에 유니크 인덱스(Unique Index)를 반드시 추가해야 합니다.

결과적으로 DB(InnoDB 기준)는 새로운 좋아요가 하나 생성(INSERT)될 때마다 B-Tree 인덱스를 두 개(PK 인덱스, 유니크 인덱스)나 갱신해야 합니다. '좋아요'는 쓰기(Insert/Delete) 트래픽이 압도적으로 많은 도메인인데, 불필요한 단일 ID 때문에 Insert 성능이 미세하게 떨어지고 디스크 공간도 낭비됩니다.

반면, 복합키를 PK로 쓰면 PK 자체가 유니크 제약조건을 겸하므로 B-Tree 트리를 하나만 관리하면 되어 쓰기 성능이 더 좋습니다.

### 2. 조회 성능 (Clustered Index의 이점)

InnoDB(MySQL, MariaDB 등)에서 PK는 곧 데이터가 물리적으로 정렬되어 저장되는 클러스터드 인덱스(Clustered Index)입니다.

복합키 (user_id, post_id)를 PK로 지정하면, 데이터가 디스크에 user_id 순으로 1차 정렬되고, 그 안에서 post_id 순으로 정렬되어 저장됩니다.

- "특정 유저가 누른 좋아요 목록 조회": user_id를 조건으로 걸면 물리적으로 모여있는 데이터를 순차적으로 쭉 읽어오면 되기 때문에(Range Scan) 디스크 I/O 관점에서 대단히 빠릅니다.
### 3. 그렇다면 언제 복합키가 '성능 저하'를 유발할까요?

질문하신 것처럼 복합키가 성능 저하나 아키텍처의 발목을 잡는 경우는 분명히 존재합니다.

- 해당 테이블을 부모로 두는 자식 테이블이 있을 때:
- 세컨더리 인덱스(Secondary Index)의 비대화:
