---
name: create-usecase
description: 헥사고날 아키텍처에 맞는 UseCase 세트(Port, Service, Command, Adapter 연결)를 생성합니다
allowed-tools: Write, Read, Glob, Grep
paths: "**/domain/**/*.java"
---

## UseCase 생성

`$ARGUMENTS` 로 전달된 정보로 UseCase를 생성합니다.
인자 형식: `<모듈명> <동작> <도메인> [command|query]` (예: `order Cancel Order command`)

기본값은 `command`(CUD) 입니다. `query`를 지정하면 조회 UseCase를 생성합니다.

---

### 명령 UseCase (command, 기본)

다음 파일을 생성합니다:

#### 1. Inbound Port: `domain/port/in/{Action}{Domain}UseCase.java`
```java
public interface {Action}{Domain}UseCase {
    {ReturnType} execute({Action}{Domain}Command command);

    record {Action}{Domain}Command(
        // 필요한 필드
    ) {
        public {Action}{Domain}Command {
            // compact constructor: null 체크, 유효성 검증
        }
    }
}
```

#### 2. Domain Service: `domain/service/{Action}{Domain}Service.java`
```java
@UseCase
@RequiredArgsConstructor
class {Action}{Domain}Service implements {Action}{Domain}UseCase {

    private final Load{Domain}Port load{Domain}Port;
    private final Save{Domain}Port save{Domain}Port;
    // 필요한 Outbound Port만 주입

    @Override
    public {ReturnType} execute({Action}{Domain}Command command) {
        // Given-When-Then 스타일 구현
    }
}
```

#### 3. Outbound Port (필요 시): `domain/port/out/` 에 추가
- `Save{Domain}Port` — 저장
- `Load{Domain}Port` — 조회
- `{Domain}EventPort` — 이벤트 발행

#### 4. 단위 테스트: `src/test/java/.../domain/service/{Action}{Domain}ServiceTest.java`
```java
class {Action}{Domain}ServiceTest {

    private final Load{Domain}Port load{Domain}Port = mock(Load{Domain}Port.class);
    private final Save{Domain}Port save{Domain}Port = mock(Save{Domain}Port.class);

    private final {Action}{Domain}Service sut = new {Action}{Domain}Service(
        load{Domain}Port, save{Domain}Port
    );

    @DisplayName("한글로 비즈니스 의도 표현")
    @Test
    void should_{result}_when_{condition}() {
        // Given
        // When
        // Then
    }
}
```

---

### 조회 UseCase (query)

#### 1. Inbound Port: `domain/port/in/{Domain}{Action}Query.java`
```java
public interface {Domain}{Action}Query {
    {ReturnType} findById({Domain}Id id);
    Page<{ReturnType}> findAll(Pageable pageable);
}
```

#### 2. Domain Service: `domain/service/Find{Domain}Service.java`

---

### 규칙
- Command는 record + compact constructor로 입력값 검증
- 1 Service = 1 UseCase (SRP)
- Service는 Outbound Port만 주입 (다른 Service 주입 금지)
- Domain Service 테스트는 순수 단위 테스트 (Spring Context 금지)
- `@UseCase` 커스텀 어노테이션 사용 (`@Service` + `@Transactional`)
- 컬렉션은 `List.copyOf()`로 방어 복사
