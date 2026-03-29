---
description: 에러 처리 및 예외 설계 규칙. 예외 클래스와 핸들러 작성 시 적용.
globs: src/main/java/**/*.java
---

# 에러 처리 전략

## 예외 계층 구조

```
DomainException (abstract, 도메인 공통 최상위)
├── OrderNotFoundException         extends EntityNotFoundException
├── OrderAlreadyCancelledException extends BusinessRuleException
├── InsufficientStockException     extends BusinessRuleException
├── DuplicateEmailException        extends BusinessRuleException
└── ...

DomainException (abstract)
├── EntityNotFoundException (abstract)    → 404
├── BusinessRuleException (abstract)      → 409
├── InvalidInputException (abstract)      → 400
└── ExternalServiceException (abstract)   → 502
```

## 도메인 예외 기본 클래스

```java
// global/error/
public abstract class DomainException extends RuntimeException {

    private final ErrorCode errorCode;

    protected DomainException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    protected DomainException(ErrorCode errorCode, String detailMessage) {
        super(detailMessage);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}

public abstract class EntityNotFoundException extends DomainException {
    protected EntityNotFoundException(ErrorCode errorCode) {
        super(errorCode);
    }
}

public abstract class BusinessRuleException extends DomainException {
    protected BusinessRuleException(ErrorCode errorCode) {
        super(errorCode);
    }

    protected BusinessRuleException(ErrorCode errorCode, String detailMessage) {
        super(errorCode, detailMessage);
    }
}
```

## ErrorCode (도메인별 에러코드)

```java
// global/error/
public interface ErrorCode {
    String getCode();
    String getMessage();
    int getStatus();
}

// order/domain/exception/
@Getter
@RequiredArgsConstructor
public enum OrderErrorCode implements ErrorCode {

    ORDER_NOT_FOUND("ORD-001", "주문을 찾을 수 없습니다", 404),
    ORDER_ALREADY_CANCELLED("ORD-002", "이미 취소된 주문입니다", 409),
    ORDER_ITEM_EMPTY("ORD-003", "주문 항목이 비어있습니다", 400),
    ORDER_AMOUNT_MISMATCH("ORD-004", "주문 금액이 일치하지 않습니다", 409);

    private final String code;
    private final String message;
    private final int status;
}
```

### ErrorCode 규칙
- 접두어로 도메인 구분: `ORD-`, `PRD-`, `MBR-`, `PAY-`, `DLV-`
- 각 도메인 모듈이 자신의 ErrorCode enum을 정의
- `global/error/ErrorCode`는 인터페이스만 제공

## 도메인별 예외 클래스

```java
// order/domain/exception/
public class OrderNotFoundException extends EntityNotFoundException {

    public OrderNotFoundException(OrderId orderId) {
        super(OrderErrorCode.ORDER_NOT_FOUND,
              "주문을 찾을 수 없습니다. orderId=" + orderId.value());
    }
}

public class OrderAlreadyCancelledException extends BusinessRuleException {

    public OrderAlreadyCancelledException(OrderId orderId) {
        super(OrderErrorCode.ORDER_ALREADY_CANCELLED,
              "이미 취소된 주문입니다. orderId=" + orderId.value());
    }
}
```

### 예외 규칙
- 도메인 예외는 반드시 `DomainException` 하위 클래스를 상속
- 예외 생성자에 식별 가능한 컨텍스트 정보 포함 (ID, 상태 등)
- 외부에 노출되면 안 되는 민감 정보(비밀번호, 토큰)는 예외 메시지에 포함 금지
- domain 계층에서 일반 `RuntimeException`, `IllegalStateException` 직접 throw 금지

## 전역 예외 핸들러

```java
// global/error/
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 도메인 예외 → 에러코드 기반 응답
    @ExceptionHandler(DomainException.class)
    ResponseEntity<ErrorResponse> handleDomainException(DomainException e) {
        ErrorCode errorCode = e.getErrorCode();
        log.warn("[{}] {}", errorCode.getCode(), e.getMessage());

        return ResponseEntity
            .status(errorCode.getStatus())
            .body(ErrorResponse.of(errorCode, e.getMessage()));
    }

    // Bean Validation 실패
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        List<FieldErrorDetail> fieldErrors = e.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldErrorDetail(fe.getField(), fe.getDefaultMessage()))
            .toList();

        log.warn("Validation failed: {}", fieldErrors);

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse.validationError(fieldErrors));
    }

    // 예상하지 못한 서버 오류
    @ExceptionHandler(Exception.class)
    ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("Unexpected error occurred", e);

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.internalError());
    }
}
```

## ErrorResponse

```java
public record ErrorResponse(
    String code,
    String message,
    List<FieldErrorDetail> fieldErrors
) {
    public static ErrorResponse of(ErrorCode errorCode, String message) {
        return new ErrorResponse(errorCode.getCode(), message, null);
    }

    public static ErrorResponse validationError(List<FieldErrorDetail> fieldErrors) {
        return new ErrorResponse("VALIDATION_ERROR", "입력값이 올바르지 않습니다", fieldErrors);
    }

    public static ErrorResponse internalError() {
        return new ErrorResponse("INTERNAL_ERROR", "서버 내부 오류가 발생했습니다", null);
    }
}

public record FieldErrorDetail(String field, String message) {}
```

## 예외 사용 위치 원칙

| 계층 | throw 가능 여부 | 예외 처리 |
|:---|:---:|:---|
| domain.model | ✅ | 자기 검증 실패 시 DomainException throw |
| domain.service | ✅ | 비즈니스 규칙 위반 시 DomainException throw |
| adapter.in.web | ❌ catch만 | GlobalExceptionHandler가 일괄 처리 |
| adapter.out.persistence | ✅ 변환만 | JPA 예외 → DomainException 변환 (예: DataIntegrityViolation → DuplicateException) |
| adapter.out.external | ✅ 변환만 | 외부 API 예외 → ExternalServiceException 변환 |
