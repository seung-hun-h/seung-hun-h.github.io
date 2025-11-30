+++
date = '2025-10-03T12:34:30+09:00'
draft = false
title = 'SpringBoot 요청 검증 예외'
ShowToc = true
+++

SpringBoot에서 HTTP 요청 검증은 `@Valid`, `@Constraint`, `@Validated` 애너테이션을 사용해서 편리하게 적용할 수 있다. SpringBoot2.x에서는 각 방식에 따라 발생하는 예외가 달라서 예외처리의 일관성 부족, 중복 검증 등의 문제가 있었다. SpringBoot3에서는 사용자들의 피드백을 수용해 많은 개선을 이루었다. 오늘은 각 방식을 사용하는 방법과 어떤 예외가 발생하는지에 대해 알아본다.

## 프로젝트 환경
- 소스코드: https://github.com/seung-hun-h/spring-validation
- **Spring Boot**: 3.5.6
- **Java**: 21

### 검증 대상
```java
public record ContentDto(
	@Valid
	UserDto user,
	@NotBlank
	String content
) {
}

public record UserDto(
	@NotBlank String name,
	@Min(value = 10) int age
) {
}
```
## 검증 방식별 특징

### @Valid 사용

`@Valid`는 JSR-303 표준 애노테이션으로, 모델 객체(`@ModelAttribute`)와 요청 본문(`@RequestBody`) 검증에 사용한다.

```java
@RestController
@RequestMapping("/valid")
public class ValidController {
    
    @PostMapping("/request-body")
    public ResponseEntity<String> requestBody(@RequestBody @Valid ContentDto contentDto) {
        return ResponseEntity.ok("Data received: " + contentDto.content());
    }
    
    @GetMapping("/model/request-param")
    public ResponseEntity<String> requestParamModel(@Valid UserDto userDto) {
        return ResponseEntity.ok("Hello " + userDto.name() + ", your age is " + userDto.age());
    }
}
```
**발생 예외:**
- 모델 객체, 요청 본문: `MethodArgumentNotValidException`

그리고 `@Valid`는 중첩 객체 검증을 위해 사용한다. `ContentDto` 내부의 `UserDto`도 함께 검증하기 위해서 사용할 수 있다.

### 제약조건(@Constraint)만 사용 (Spring 6.1+)

Spring 6.1부터는 `@Valid` 없이도 개별 파라미터(`@RequestParam`)에 제약조건을 직접 적용할 수 있다.

```java
@RestController
@RequestMapping("/constraints")
public class ConstraintsController {
    
    @GetMapping("/request-param")
    public ResponseEntity<String> requestParam(
        @RequestParam @Min(10) int age,
        @RequestParam @NotBlank String name
    ) {
        return ResponseEntity.ok("Hello " + name + ", your age is " + age);
    }
}
```
**발생 예외:**
- 개별 파라미터: `HandlerMethodValidationException`

**특징:** 개별 파라미터는 검증되지만, 모델 객체 내부 검증은 적용되지 않는다.

### @Validated 사용

`@Validated`는 Spring 전용 애노테이션으로 AOP 기반으로 동작한다. Spring 6.1부터는 built-in method validation이 지원되어 더 이상 필요하지 않다.
개별 파라미터는 `@Constraint`만 사용하면되고, 모델 객체 검증도 별다른 작업을 하지 않아도 검증 가능 하다. 요청 본문은 `@Validated`, `@Valid`을 추가해줘야 한다.

```java
@RestController
@RequestMapping("/validated")
@Validated // 클래스 레벨에서 AOP 활성화
public class ValidatedController {
    
    @GetMapping("/request-param")
    public ResponseEntity<String> requestParam(
        @RequestParam @Min(10) int age,
        @RequestParam @NotBlank String name
    ) {
        return ResponseEntity.ok("Hello " + name + ", your age is " + age);
    }

	@GetMapping("/model/request-param")
	public ResponseEntity<String> requestParamModel(
		UserDto userDto
	) {
		return ResponseEntity.ok("Hello " + userDto.name() + ", your age is " + userDto.age());
	}

	@PostMapping("/request-body")
	public ResponseEntity<String> requestBodyContent(
		@Validated @RequestBody ContentDto contentDto
	) {
		UserDto userDto = contentDto.user();

		return ResponseEntity.ok("Hello " + userDto.name() + ", your age is " + userDto.age() + ". content: " + contentDto.content());
	}
}
```

**발생 예외:**
- 개발 파라미터: `ConstraintViolationException`
- 모델 객체, 요청 본문: `MethodArgumentNotValidException`

**제한사항:**
- AOP 프록시 기반으로 동작하여 성능 오버헤드 존재
- 모델 객체 내부 검증 규칙은 적용되지 않음
- Spring 6.1+ 환경에서는 built-in method validation 사용 권장

### 정리
이제 개별 파라미터, 모델 객체, 요청 본문을 검증하기 위해서는 AOP 방식인 `@Validated`는 지양하고 `@Constraint` `@Valid`만 사용하자.

## HTTP 요청 관련 예외
요청 검증 외 아래와 같은 다양한 요청 관련 예외가 발생할 수 있다

**발생할 수 있는 예외:**
- `MissingServletRequestParameterException`: 필수 파라미터 누락
- `MethodArgumentTypeMismatchException`: 타입 변환 실패
- `HttpMessageNotReadableException`: 잘못된 JSON 형식 또는 빈 요청 본문
- `HttpMediaTypeNotSupportedException`: Content-Type 불일치

## 참고 자료
- [Spring Framework Validation 공식 문서](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html)
- [Spring Framework 6.1 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes)

