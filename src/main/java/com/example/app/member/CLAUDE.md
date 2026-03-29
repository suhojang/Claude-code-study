# 회원 모듈 (Member Module)

**소유팀**: team-member
**테이블 접두어**: `mbr_`
**API 경로**: `/api/v1/members`, `/api/v1/auth`

## 모듈 구조

```
member/
├── MemberModuleConfig.java
├── api/                            # ✅ 공개 API
│   ├── FindMemberQuery.java        #   회원 조회 공개 인터페이스 (동기)
│   ├── MemberRegisteredEvent.java
│   └── MemberDeletedEvent.java
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Member.java
│   │   ├── MemberId.java
│   │   ├── Email.java              # VO: 이메일 형식 검증
│   │   ├── Password.java           # VO: 암호화된 비밀번호
│   │   ├── MemberRole.java         # enum: USER, SELLER, ADMIN
│   │   └── ShippingAddress.java    # VO: 배송 주소
│   ├── port/in/
│   │   ├── RegisterMemberUseCase.java
│   │   ├── LoginUseCase.java
│   │   ├── UpdateProfileUseCase.java
│   │   └── MemberFindQuery.java
│   ├── port/out/
│   │   ├── SaveMemberPort.java
│   │   ├── LoadMemberPort.java
│   │   ├── MemberEventPort.java
│   │   └── PasswordEncoderPort.java
│   ├── service/
│   │   ├── RegisterMemberService.java
│   │   ├── LoginService.java
│   │   ├── UpdateProfileService.java
│   │   └── FindMemberService.java
│   └── exception/
│       ├── MemberErrorCode.java     # MBR-001 ~ MBR-xxx
│       ├── MemberNotFoundException.java
│       ├── DuplicateEmailException.java
│       └── InvalidPasswordException.java
└── adapter/
    ├── in/web/
    │   ├── MemberController.java
    │   ├── AuthController.java
    │   ├── RegisterMemberRequest.java
    │   ├── LoginRequest.java
    │   └── MemberResponse.java
    └── out/
        ├── persistence/
        │   ├── MemberPersistenceAdapter.java
        │   ├── MemberJpaEntity.java
        │   ├── MemberJpaRepository.java
        │   └── MemberPersistenceMapper.java
        └── event/
            └── MemberEventPublisherAdapter.java
```

## 공개 API: FindMemberQuery

```java
public interface FindMemberQuery {
    MemberInfo findById(Long memberId);

    record MemberInfo(
        Long id,
        String name,
        String email,
        String role
    ) {}
}
```

## 발행/수신 이벤트

| 발행 이벤트 | 시점 | 소비 모듈 |
|:---|:---|:---|
| `MemberRegisteredEvent` | 회원 가입 완료 | notification (환영 메일) |
| `MemberDeletedEvent` | 회원 탈퇴 | order (미완료 주문 처리), notification |

## 비즈니스 규칙

- 이메일 중복 불가
- 비밀번호: 최소 8자, 영문+숫자+특수문자
- 비밀번호는 반드시 암호화하여 저장 (PasswordEncoderPort)
- 로그인 실패 5회 시 계정 잠금

## 보안 주의사항

- 비밀번호, 토큰은 로그/예외 메시지에 절대 포함 금지
- `Password` VO의 `toString()`은 마스킹 처리

## 테스트 실행

```bash
./gradlew test --tests "*.member.*"
```
