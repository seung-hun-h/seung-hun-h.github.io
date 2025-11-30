+++
date = '2025-10-03T12:52:21+09:00'
draft = true
title = 'SpringMVC Error Responses'
+++
# Spring MVC 에러 응답 개선: ProblemDetail 도입 배경과 GitHub 이슈 정리

## 1. 왜 에러 응답을 다시 생각해야 할까?

### 기존 Spring Boot 기본 에러 응답의 한계
- `timestamp`, `status`, `error`, `path` 정도만 제공.
- 어떤 값이 잘못됐는지 알 수 없음.
- 예외 종류마다 응답 구조가 제각각이라 API 일관성이 떨어짐.
- MVC, WebFlux, Boot 간 에러 처리 시스템도 서로 다름.
- 클라이언트는 같은 팀 서비스라도 다른 JSON 구조를 처리해야 하는 문제 발생.

### 특히 “MVC vs WebFlux” 간 구조 차이는 큰 문제
같은 팀에서 개발하더라도 Spring MVC와 WebFlux는 기본 에러 구조가 다르다.

- **Spring MVC**  
  Boot ErrorAttributes 기반 JSON
- **Spring WebFlux**  
  자체 ErrorWebExceptionHandler 기반 JSON

결과적으로:
- 서비스마다 에러 JSON 구조가 다르게 나타남
- API 문서, 클라이언트 코드, 모니터링 규칙을 통일할 수 없음
- 팀 내부 합의가 깨지고 생산성 저하

Spring 팀도 이 문제를 공식적으로 인지했고,  
ProblemDetail 도입의 핵심 이유 중 하나가 되었다.

---

## 2. Spring GitHub 이슈에서 논의된 핵심 내용

아래 이슈들은 **ProblemDetail 도입을 이해하는 데 필수적인 논의 흐름**이다.

### 2.1 Spring Boot Issue #19525
**"Consider using RFC 7807 problem details for error responses"**  
https://github.com/spring-projects/spring-boot/issues/19525

핵심 요약:
- Boot 기본 에러 JSON은 Spring 고유 포맷 → 표준 아님.
- 업계는 RFC 7807 표준을 많이 사용 중.
- 표준 기반 에러 모델 도입 필요성이 공식적으로 제기됨.

---

### 2.2 Spring Framework Issue #27052
**"Support for problem details based on RFC 7807"**  
https://github.com/spring-projects/sspring-framework/issues/27052

핵심 요약:
- MVC/WebFlux/Boot 간 에러 처리 시스템을 하나의 표준 구조로 통합해야 한다.
- Framework 6에서 에러 모델 전면 개편.
- 이후 ProblemDetail을 중심으로 Spring 에러 처리 체계가 재정립됨.

---

### 2.3 Spring Framework Issue #28187
**"Add types to represent RFC 7807 problem details and exceptions"**  
https://github.com/spring-projects/spring-framework/issues/28187

핵심 요약:
- `ProblemDetail`, `ErrorResponse`, `ErrorResponseException` 정의.
- 모든 HTTP 오류를 표준 모델로 포장하기 위한 기반 타입 설계.

---

### 2.4 Spring Framework Issue #28665
**"Allow dynamic properties in ProblemDetail"**  
https://github.com/spring-projects/spring-framework/issues/28665

핵심 요약:
- ProblemDetail에 커스텀 필드를 동적으로 추가 가능.
- 서비스별 errorCode, userId, redirectUrl 등 확장성 확보.

---

### 2.5 Spring Framework Issue #28814
**"Allow for external customization and i18n of the detail"**  
https://github.com/spring-projects/spring-framework/issues/28814

---

### 2.6 Spring Framework Issue #30566
**"Allow setting the ProblemDetail.type via a MessageSource"**  
https://github.com/spring-projects/spring-framework/issues/30566

핵심 요약:
- title/detail/type 메시지를 MessageSource 기반으로 국제화 가능.

---

### 2.7 Spring Framework Issue #28189
**"Support application/problem+json as Content-Type"**  
https://github.com/spring-projects/spring-framework/issues/28189

핵심 요약:
- ProblemDetail JSON을 RFC 7807 공식 미디어 타입으로 제공.

---

### 2.8 Spring Boot Issue #32634
**"Auto-configure ProblemDetails support for Spring MVC and WebFlux"**  
https://github.com/spring-projects/spring-boot/issues/32634

핵심 요약:
- Boot 3에서 ProblemDetail 자동 구성이 기본 제공됨.
- 개발자는 설정 없이 바로 ProblemDetail 기반 에러 응답 사용 가능.

---

## 3. ProblemDetail이 왜 중요한가?

### Spring 생태계 전체의 “에러 응답 표준화”
- MVC / WebFlux / Boot 간 JSON 응답 구조를 하나로 통일.
- API 문서(Swagger), 프론트 처리, 로그 구조가 전부 단순해짐.
- RFC 7807 기반 표준 모델 → 생태계 호환성 대폭 향상.
- properties 기반 확장성 보장.
- MessageSource 통한 국제화(i18n)까지 지원.

### 개발자가 얻는 실질적 이점
- 모든 에러가 동일한 구조 → 클라이언트 코드 단순화.
- 비즈니스 에러(BusinessException)도 표준 구조로 반환 가능.
- 테스트, API 문서화, 모니터링 강화.
- 팀 내 개발 규약을 간단히 정의할 수 있음.

---

## 4. 학습 및 실험 과정에서의 핵심 포인트

### 4.1 ProblemDetail 기본 활성화
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

4.2 @ExceptionHandler가 동작하지 않은 이유

Spring MVC 기본 핸들러(problemDetailsExceptionHandler)가
exceptionHandlerAdviceCache에서 먼저 매칭되었기 때문.

→ 커스텀 핸들러가 실행되지 않음.

해결책:

@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)

4.3 검증 에러(MethodArgumentNotValidException) 커스터마이징

ProblemDetail에 errors[] 배열 추가해 필드별 검증 오류 제공.

표준 필드(type, title, status, detail, instance) 기반을 유지하면서
서비스별 확장 정보만 properties에 추가.

5. 글에서 강조할 핵심 메시지

ProblemDetail은 단순한 기능이 아니라
Spring MVC / WebFlux / Boot 전체 에러 응답을 표준화하기 위한 전략적 결정이다.

Spring은 GitHub 이슈를 통해 3년 넘게 이 문제를 논의해왔고
여러 버전을 거치며 프레임워크 전반을 재설계했다.

개발자는 이제 DTO 커스터마이징보다
ProblemDetail 표준 기반 + properties 확장 방식을 사용하는 것이 적합하다.

예외 처리 우선순위(@Order), ErrorResponse, ProblemDetail 내부 구조를 이해하면
실무에서 에러 응답을 안정적으로 설계할 수 있다.

