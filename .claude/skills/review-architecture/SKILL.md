---
name: review-architecture
description: 현재 코드의 Spring Modulith + Hexagonal 아키텍처 규칙 준수 여부를 전수 검사합니다
context: fork
allowed-tools: Read, Grep, Glob
model: sonnet
---

## 아키텍처 규칙 준수 검증

프로젝트의 모든 Java 소스 파일을 검사하여 아키텍처 위반 사항을 보고합니다.

### 검사 항목 1: 모듈 경계 위반

**다른 모듈의 내부 패키지 import 검사:**
- `com.example.app.{다른모듈}.domain.*` import → 위반
- `com.example.app.{다른모듈}.adapter.*` import → 위반
- `com.example.app.{다른모듈}.api.*` import → 허용
- `com.example.app._shared.*` import → 허용

**검사 방법:**
```
각 Java 파일의 import 문을 읽고, 파일이 속한 모듈과 import 대상 모듈이 다른 경우
대상이 api/ 또는 _shared/가 아니면 위반으로 기록
```

### 검사 항목 2: 헥사고날 아키텍처 위반

- **domain/model/** 에 프레임워크 어노테이션 존재 여부
  - `@Entity`, `@Table`, `@Id`, `@Column` → 위반
  - `@Getter`, `@Setter`, `@Data`, `@Builder` → 위반
  - `@Service`, `@Component`, `@Repository` → 위반
- **domain/ → adapter/ 역방향 의존** 여부
  - domain 패키지 파일에서 adapter 패키지 import → 위반
- **Controller에 비즈니스 로직** 존재 여부
  - Controller에서 Repository 직접 주입 → 위반
  - Controller에서 조건 분기/계산 로직 → 위반 (DTO 변환과 UseCase 호출만 허용)

### 검사 항목 3: 네이밍 규칙 위반

| 대상 | 규칙 | 검사 |
|:---|:---|:---|
| UseCase (명령) | `{Action}{Domain}UseCase` | 인터페이스 이름 패턴 |
| UseCase (조회) | `{Domain}{Action}Query` | 인터페이스 이름 패턴 |
| Service | `{Action}{Domain}Service` | 클래스 이름 패턴 |
| Outbound Port | `Save{Domain}Port`, `Load{Domain}Port` | 인터페이스 이름 패턴 |
| JPA Entity | `{Domain}JpaEntity` | `JpaEntity` 접미어 |
| Mapper | `{Domain}PersistenceMapper` | `PersistenceMapper` 접미어 |
| Controller | `{Domain}Controller` | `Controller` 접미어 |
| 이벤트 | `{Domain}{Action}Event` | 이벤트 record 이름 |
| 예외 | `{Domain}{상황}Exception` | 예외 클래스 이름 |

### 검사 항목 4: 영속성 규칙 위반

- JPA Entity에 `@Setter` 사용 여부 → 위반
- Repository가 `public` 접근 제한자인지 → 위반 (package-private이어야 함)
- 다른 모듈 테이블 JOIN 쿼리 존재 여부

### 검사 항목 5: 테스트 규칙 위반

- `domain/service/` 테스트에 `@SpringBootTest` 사용 → 위반 (순수 단위 테스트여야 함)
- `domain/model/` 테스트에 Spring Context 로드 → 위반

### 결과 보고 형식

```markdown
## 아키텍처 검증 결과

### 요약
- 검사 파일 수: N개
- 위반 건수: N건 (Critical: N, Warning: N)

### 위반 상세

| # | 파일 | 위반 유형 | 설명 | 심각도 |
|:--|:-----|:---------|:-----|:------|
| 1 | order/domain/service/Foo.java:15 | 모듈 경계 위반 | product.domain.model.Product import | Critical |
| 2 | order/domain/model/Order.java:3 | 헥사고날 위반 | @Entity 어노테이션 사용 | Critical |
| 3 | order/adapter/out/persistence/OrderJpaRepository.java:1 | 가시성 위반 | public interface (package-private이어야 함) | Warning |

### 권장 조치
1. ...
2. ...
```
