---
name: db-migration
description: 모듈별 Flyway 마이그레이션 SQL 파일을 규칙에 맞게 생성합니다
allowed-tools: Write, Read, Glob, Bash
---

## Flyway 마이그레이션 생성

`$ARGUMENTS` 로 전달된 정보로 마이그레이션 파일을 생성합니다.
인자 형식: `<모듈명> <설명>` (예: `order create_order_tables`)

### 파일 경로 규칙

```
src/main/resources/db/migration/{module}/V{yyyyMMdd}{nn}__{description}.sql
```

- **날짜**: 오늘 날짜 (yyyyMMdd 형식)
- **nn**: 해당 모듈 디렉토리 내 같은 날짜의 기존 파일 다음 번호 (01부터 시작)
- **description**: snake_case, 간결한 영문 설명

### 실행 절차

1. 기존 마이그레이션 파일 목록 확인: `ls src/main/resources/db/migration/{module}/`
2. 다음 버전 번호 결정
3. SQL 파일 생성

### 테이블 접두어 규칙

| 모듈 | 접두어 |
|:---|:---|
| order | `ord_` |
| product | `prd_` |
| member | `mbr_` |
| payment | `pay_` |
| delivery | `dlv_` |
| notification | `ntf_` |

### SQL 작성 규칙

```sql
-- V2026032901__create_order_tables.sql

CREATE TABLE ord_order (
    order_id    BIGSERIAL PRIMARY KEY,
    member_id   BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    total_amount BIGINT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP NOT NULL DEFAULT now()
);

-- 인덱스
CREATE INDEX idx_ord_order_member_id ON ord_order(member_id);
CREATE INDEX idx_ord_order_status ON ord_order(status);
```

### 금지 사항
- 기존 마이그레이션 파일 수정 금지 (새 파일로 ALTER)
- 다른 모듈 테이블에 FOREIGN KEY 금지
- 다른 모듈의 마이그레이션 디렉토리에 파일 생성 금지
- 컬럼명은 snake_case
- `NOT NULL` 제약은 비즈니스 규칙에 따라 적절히 적용
- ENUM 타입 대신 VARCHAR + 애플리케이션 레벨 검증 사용
