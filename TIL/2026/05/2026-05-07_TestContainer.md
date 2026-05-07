# TestContainer

> 날짜: 2026-05-07
> 원본 노션: [링크](https://www.notion.so/TestContainer-358edfd5a4768085b555ee214a55dc94)

---

### TestContainer 중복 실행으로 인한 충돌 문제

기존에는 test profile 설정 후 yml 파일에서 test용 설정 X

- 기존 compose.yaml 컨테이너랑 충돌 문제 발생
```java
@ActiveProfiles("test")
public abstract class AbstractIntegrationTest {
```

- application.yml 에 test 용 profile 설정 추가
- jdbc 지원 하는 db는 application.yaml 파일에서 아래와 같은 설정 하면 AbstractIntegrationTest 클래스에서 따로 컨테이너 설정 안 해도 됨
```yaml
spring:
  config:
    activate:
      on-profile: test
  jpa:
    hibernate:
      ddl-auto: create-drop
  datasource:
    url: jdbc:tc:mariadb:11.4:///test?TC_REUSABLE=true
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
```

- ?TC_REUSABLE=true ?
### @ServiceConnection  vs 
@Container와 @DynamicPropertySource

아주 예리한 질문입니다! 두 코드는 Testcontainers를 다루는 "과거의 표준 방식(위)"과 "현재의 최신 방식(아래)"을 정확하게 보여주고 있습니다.

핵심부터 말씀드리면, @ServiceConnection은 스프링 부트 3.1부터 도입된 엄청난 "마법의 어노테이션"입니다. 이 녀석이 있으면 @Container와 @DynamicPropertySource가 하던 귀찮은 일들을 한 방에 해결해 줍니다.

차이점을 명확하게 비교해 드릴게요!

---

### 1. 첫 번째 코드 (기존 방식: JUnit이 관리)

Java

# 

@Testcontainers // 1. "JUnit아, 이 클래스에 있는 컨테이너들 좀 띄워줘!"
public abstract class AbstractIntegrationTest {

    @Container // 2. "이 Redis 컨테이너를 테스트 시작 전에 띄우고, 끝나면 종료해!"
    protected static final GenericContainer<?> redisContainer = ...

    @DynamicPropertySource // 3. "컨테이너 떴지? 거기 랜덤 포트 몇 번이야? 스프링 설정에 내가 직접 꽂아 넣을게!"
    static void configureProperties(DynamicPropertyRegistry registry) { ... }
}

- 주체: 테스트 프레임워크인 JUnit이 컨테이너의 생명주기(시작/종료)를 관리합니다.
- 특징: 스프링은 도커 컨테이너가 떴는지 안 떴는지 모릅니다. 그래서 개발자가 직접 @DynamicPropertySource를 써서 "Redis 호스트는 이거고, 포트는 이거야"라고 스프링 환경 변수에 일일이 주입해 주어야 합니다.
### 2. 두 번째 코드 (최신 방식: Spring Boot가 관리)

Java

# 

// @Testcontainers 없음! (JUnit이 관리 안 함)
public abstract class AbstractIntegrationTest {

    @ServiceConnection // 1. "스프링 부트야, 네가 알아서 이 Elasticsearch 컨테이너 띄우고, 포트 알아내서 설정까지 다 해줘!"
    protected final static ElasticsearchContainer ELASTICSEARCH = ...
}

- 주체: Spring Boot가 직접 컨테이너를 관리합니다.
- 왜 @Container가 없나요?: @ServiceConnection이 붙어있으면, 스프링 부트가 컨테이너를 발견하고 알아서 실행(start)시킵니다. 따라서 JUnit에게 실행을 맡기는 @Container와 @Testcontainers가 아예 필요 없습니다.
- 왜 @DynamicPropertySource가 없나요?: 스프링 부트가 Elasticsearch 컨테이너임을 인식하고, "아, 이거 엘라스틱서치구나? 그럼 포트 알아내서 spring.elasticsearch.uris에 내가 알아서 세팅할게" 하고 자동으로 설정을 주입해 버립니다.
---

### 🧐 그런데 두 번째 코드 아래에 있는 SpiceDB는 왜 다르죠?

두 번째 코드를 자세히 보시면 Elasticsearch는 @ServiceConnection을 썼는데, 그 아래 SpiceDB는 수동으로 하고 있죠? 여기에는 아주 중요한 이유가 있습니다.

Java

# 

    // SpiceDB는 @ServiceConnection을 못 씁니다!
    protected static GenericContainer<?> spicedbContainer = ...

    static {
        spicedbContainer.start(); // 직접 컨테이너를 시작함 (@Container 대용)
    }

    @DynamicPropertySource // 직접 속성을 주입함
    protected static void registerSpiceDBProperties(...) { ... }

- @ServiceConnection은 Redis, MariaDB, Elasticsearch, Kafka 같이 스프링 부트가 공식적으로 지원하는 유명한 서비스들만 알아서 처리해 줄 수 있습니다.
- SpiceDB는 스프링 부트가 모르는 녀석입니다. 그래서 @ServiceConnection을 달아놔도 스프링이 "이게 뭐지? 설정 어떻게 하는지 모르는데?" 하고 무시합니다.
- 따라서 엘라스틱서치는 최신 방식(@ServiceConnection)으로 꿀을 빨고, 스프링이 모르는 SpiceDB는 어쩔 수 없이 수동 방식(static { start() }와 @DynamicPropertySource)을 섞어서 쓴 것입니다.
### 💡 요약

- @Container + @DynamicPropertySource: 구식. 내가 직접 컨테이너 띄우고, 포트 번호 캐내서 스프링에 일일이 알려주는 수동 방식.
- @ServiceConnection: 신식 (Spring Boot 3.1+). 스프링이 알아서 띄우고 설정까지 싹 다 해주는 자동 방식. 지원되는 DB/서비스라면 무조건 이걸 쓰는 게 이득입니다! (현재 첫 번째 코드의 Redis도 @ServiceConnection으로 바꿀 수 있습니다!)
