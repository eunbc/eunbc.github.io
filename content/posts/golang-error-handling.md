---
title: "Go 에러 핸들링 개선기 - 단순함에 Go며들었다.."
date: 2025-05-29T09:30:00+09:00
draft: false
tags: ["go", "error-handling", "backend"]
categories: ["go"]
summary: "Go의 에러 처리 철학(에러는 값이다, 명시적 처리)을 짚고, 에러 래핑·센티넬 에러·errors.Is/As·전역 ErrorHandler를 활용해 API 서버의 에러 핸들링 구조를 개선한 과정 공유."
slug: "golang-error-handling-improvement"
ShowToc: true
TocOpen: false
cover:
  image: "/images/golang-error-handling/9fc5b5921682d99a.png"
  alt: "Go 에러 핸들링 개선기"
  relative: false
  hidden: true
---

안녕하세요! 레인입니다 ☔️

현재 사내에서 Go를 주력 언어로 사용중이고, 최근 Go 서버의 에러 핸들링 구조 개선 작업을 진행했습니다. Go의 에러 처리 개념부터 실제 API 서버에 적용한 방법까지 공유드리고자 합니다.

![스크린샷 2025-05-28 오후 12.38.05.png](/images/golang-error-handling/9fc5b5921682d99a.png)

귀여운 Go의 마스코트, gopher와 함께 시작해볼까요! Let’s Go~

## 1. Go에서 에러란?

Go에서 에러는 **비정상 상태를 나타내는 값**입니다.

## 2. Go 에러 특징

### 2-1. 에러는 값이다

```go
func AddUser(username string, password string) (id int, err error) {
    id, err = userMysqlRepository.Save(username) 
    if err != nil {
        return 0, err // 실패: zero value, err 리턴
    }
    retrun id, nil // 성공: id와 nil 반환
}
```

회원가입을 하는 간단한 예시입니다. err가 nil이면 정상, nil이 아니면 실패로 처리하는 게 기본 흐름입니다.

Go에서는 에러도 일반 값처럼 다루기 때문에 명시적으로 다루는 패턴이 기본입니다.

\*nil은 다른 언어에서 표현하는 null과 같습니다

### 2-2. 성공했다면 결과를 신뢰할 수 있어야 한다

<table>
<thead>
<tr><th>Good</th><th>Bad</th></tr>
</thead>
<tbody>
<tr>
<td>

```go
func SignUp() {
    id, err := AddUser("rain.cho", "gbike123")
    if err != nil {
        // fail
    }
    // success: 신뢰할 수 있는 id
}
```

</td>
<td>

```go
func SignUp() {
    id, err := AddUser("rain.cho", "gbike123")
    if err != nil {
        // fail
    }

    if id == 0 {
        // 신뢰할 수 없는 id?
        // fail -> 혼란을 유발한다!
    }

    // success
}
```

</td>
</tr>
</tbody>
</table>

성공했다면 err가 nil인 상태이고, 결과값을 신뢰할 수 있는 상태여야 합니다.

반대로 실패했다면 err가 nil이 아니고, 실패 처리를 해야 합니다.

err가 nil이면서 결과값을 다시 검증하는 모호한 로직을 지양하는게 좋습니다.

### 2-3. error는 인터페이스 타입이다

```go
type error interface {
    Error() string
}
```

에러는 인터페이스 타입입니다. 따라서 Error() string을 구현하는 타입은 얼마든지 error로 사용할 수 있습니다.

여러분의 주력언어와 어떤 차이점이 있나요?

단순**.** 심플**.**명확 이것이 바로 Go의 매력입니다 :)

이제 여기부터는 **Golang**으로 서버 개발을 하시는 분 **&** 할 의향이 있는 분들 대상입니다**!**

## 3. Go API 서버에서 에러 핸들링을 어떻게 해야할까?

[google styleguide](https://google.github.io/styleguide/go/decisions#handle-errors)에 따르면, 에러는 다음과 같이 처리할 수 있습니다.

- 에러를 즉시 처리하고 해결한다

- 에러를 호출자에게 반환한다

- 예외적인 상황에서는 log.Fatal을 호출하거나 (정말 필요할 경우에만) panic을 사용한다

### 3-1. 에러 래핑

API 서버 내부에서 발생하는 에러는 즉시 처리하는 경우를 제외하고 대부분 호출자에게 반환하게 됩니다. 에러를 호출자에게 반환할 때, 어디서 발생했는지를 명확히 하려면 메시지를 추가하는 것이 좋습니다.

fmt.Errorf(“msg : %w”, err) 를 사용하면 간단히 래핑 가능합니다.

```go
func AddUser(username string, password string) (id int, err error) {
  id, err = userMysqlRepository.Save(username) 
  if err != nil {
    return 0, fmt.Errorf("userMysqlRepository.Save: %w", err) 
  }
  retrun id, nil 
}

// 에러 래핑을 하지 않을 경우의 출력 결과
Error 1054 (42S22): Unknown column 'asdf' in 'field list'

// 에러 래핑을 했을 경우의 출력 결과
userMysqlRepository.Save: Error 1054 (42S22): Unknown column 'asdf' in 'field list'
```

#### Tip) pkg/errors는 더이상은… naver…

온라인에 존재하는 많은 자료에서 pkg/errors 를 많이 보여주고 있는데요, [pkg/errors](https://github.com/pkg/errors)는 21년 12월에 아카이브 된 레포지토리이므로 새로 작성하는 코드에서는 사용하지 않는 것이 좋습니다.

이를 대신하여 Go의 표준 패키지를 사용할 수 있습니다.

|      |                |                                                      |
|------|----------------|------------------------------------------------------|
| 기능 | **pkg/errors** | **Go 표준 패키지**                                   |
| 래핑 | errors.Wrap    | fmt.Errorf(“msg: %w”, err) (Go 1.13+)                |
| 생성 | errors.New     | errors.New (표시 형식은 같지만 다른 패키지임에 유의) |

#### Tip) 메시지 작성시 failed to는 생략

failed to 같은 메시지는 중복되므로 생략하여 간결하게 나타내면 좋습니다

<table>
<thead>
<tr><th>Bad</th><th>Good</th></tr>
</thead>
<tbody>
<tr>
<td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %w", err)
}
```

</td>
<td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %w", err)
}
```

</td>
</tr>
<tr>
<td><code>failed to x: failed to y: failed to create new store: the error</code></td>
<td><code>x: y: new store: the error</code></td>
</tr>
</tbody>
</table>

### 3-2. 센티넬 에러 활용하기

의도적으로 에러를 던져야 하는 경우, 센티넬 에러를 활용해보는건 어떨까요? 센티넬 에러는 패키지 수준의 전역 변수로 선언됩니다. 상태가 명확하게 구분될 경우, 에러로 분기 처리를 할 경우 사용하기에 적합합니다.

변수명은 Err로 시작하는 것이 관례입니다. 또한 에러 메시지는 소문자로 시작하되, 마침표(.) 로 끝나지 않는 것 또한 관례입니다.

```go
var ErrParameterRequired = errors.New("parameter required")

// sql.ErrNoRows
var ErrNoRows = errors.New("sql: no rows in result set")

// io.EOF
// 대표적으로 관례를 따르지 않는 예외도 있긴 합니다 ^^;
var EOF = errors.New("EOF")

// context.Canceled
var Canceled = errors.New("context canceled")
```

### 3-3. errors.As, Is 활용하기 (Go 1.13+)

- [errors.Is](http://errors.Is) : 에러가 특정 에러 값과 같은지 확인할 수 있습니다. 내부적으로 Unwrap()을 재귀적으로 호출하며, 에러 체인 중에 동등한 값(==) 이 있는지를 검사합니다. 주로 센티넬 에러(os.ErrNotExist 등)와 비교할 때 사용합니다.

```go
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("파일이 존재하지 않음")
}
```

- errors.As : 에러가 특정 에러 타입으로 변환 가능한지 확인하고, 해당 타입으로 언래핑할 수 있게 해줍니다.

- 내부적으로 Unwrap()을 따라가며, 에러 체인 중 원하는 타입이 있는지 확인합니다. 주로 구조체 타입 에러에서 세부 필드를 참조할 때 사용합니다.

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("경로 에러:", pathErr.Path)
}
```

### 3-4. 전역 ErrorHandler 활용

웹 프레임워크 Fiber에서는 기본적으로 ErrorHandler를 제공합니다. 공통 포맷의 JSON으로 응답하도록 구성하는 등 프로젝트에 맞게 바꾸어서 사용하면 에러 처리의 효율성을 높일 수 있습니다.

[https://docs.gofiber.io/guide/error-handling](https://docs.gofiber.io/guide/error-handling)

```go
ErrorHandler: func(ctx *fiber.Ctx, err error) error {
    if errors.Is(err, apperror.ErrParsingError) {
        return ctx.Status(400).JSON(...)
    }
    ...
}
```

## 4. API 서버 Before → After

그래서 이 모든것을 종합하여 서버를 어떻게 리팩토링 했냐면…!

처음 보는 사람도 읽기 쉬운 코드이면서, 유지보수성을 높일 수 있는 방향으로 설계하고자 하였습니다.

### 4-1. BEFORE

```go
func (s StatusController) GetStatus(c *fiber.Ctx) (err error) {
    params := new(resource.GetStatusRequest)
    if err := c.QueryParser(params); err != nil {
        return errorResponse(c, fiber.StatusOK, utils.API_RESPONSE_FAIL_CODE, utils.API_RESPONSE_ERROR_MESSAGE)
    }
    logger.Logger.Debug("get status  request params parser :: ", params)

    if err := validator.New().Struct(params); err != nil {
        return errorResponse(c, fiber.StatusOK, utils.API_RESPONSE_FAIL_CODE, utils.API_RESPONSE_ERROR_MESSAGE)
    }

    if params.LockSn == nil && params.BicycleSn == nil {
        return errorResponse(c, fiber.StatusOK, utils.API_RESPONSE_FAIL_CODE, utils.API_RESPONSE_BAD_REQUEST_MESSAGE)
    }

    result, err := s.StatusUsecase.Status(c.Context(), *params)

    if err != nil {
        return errorResponse(c, fiber.StatusOK, utils.API_RESPONSE_FAIL_CODE, utils.API_RESPONSE_ERROR_MESSAGE)
    }

    if result == nil {
        return errorResponse(c, fiber.StatusOK, utils.API_RESPONSE_SUCCESS_CODE, utils.API_RESPONSE_SUCCESS_MESSAGE)
    }

    return successResponse(c, result)
}

func errorResponse(c *fiber.Ctx, status int, result string, message string) error {
    return c.Status(status).JSON(resource.CommonResponse{
        Result:  result,
        Message: message,
        Data:    []string{},
    })
}
```

**기존 버전의 문제점**

- 로깅 없음, 에러 추적 불가

- 일관되지 못한 에러 핸들링, 성공/실패의 구분 모호

- 컨트롤러가 많은 책임을 지고 있음

- HTTP 상태 코드, 에러 메시지를 변경하고 싶을 경우 일일이 변경해줘야 하는 번거로움

### 4-2. AFTER

**CONTROLLER**

```go
func (s StatusController) GetStatus(c *fiber.Ctx) error {
    params := new(request.GetStatusRequest)
    if err := c.QueryParser(params); err != nil {
        return ErrPasingFailed
    }

    if err := validator.New().Struct(params); err != nil {
        return ErrValidationFailed
    }
	
    result, err := s.StatusUsecase.Status(c.Context(), *params)
    if err != nil {
        return fmt.Errorf("StatusUsecase.Status: %w", err)
    }

    return successResponse(c, result)
}
```

- 컨트롤러에서는 에러를 받아서 래핑만 해주고 리턴, 책임 간소화

- 성공/실패를 명확히 구분

**ERROR_HANDLER**

```go
  ErrorHandler: func(ctx *fiber.Ctx, err error) error {
      var status int
      var userMsg string
      switch {
      case errors.Is(err, apperror.ErrValidationFailed):
          status = fiber.StatusBadRequest
          userMsg = "validation error"
          logger.Logger.Warn(err.Error())

      case errors.Is(err, apperror.ErrParsingFailed):
          status = fiber.StatusBadRequest
          userMsg = "parsing error"
          logger.Logger.Warn(err.Error())

      case errors.Is(err, apperror.ErrVehicleNotFound):
          status = fiber.StatusInternalServerError
          userMsg = "vehicle not found"
          logger.Logger.Warn(err.Error())

      default:
          status = fiber.StatusInternalServerError
          userMsg = "internal server error"
          logger.Logger.Errorf("%v", err)
      }

      return ctx.Status(status).JSON(response.CommonApiResponse{
          Code:    status,
          Message: userMsg,
          Data:    nil,
      })
  },
```

- 센티넬 에러 + [errors.Is](http://errors.Is)를 활용하여, 에러별 HTTP 상태코드, 응답 메시지, 로그 레벨, 로깅 방식 등을 일관되고 자유롭게 조정 가능

**SENTINEL_ERROR**

```go
// controller layer
var (
    ErrValidationFailed  = errors.New("validation fail")
    ErrParsingFailed     = errors.New("parsing fail")
)

// usecase layer
var (
    ErrVehicleNotFound = errors.New("vehicle not found")
)

// db layer
var (
    ErrNoData                     = errors.New("no data")
    ErrRedisKeyNotFound           = errors.New("redis key not found")
)

//...
```

## 5. 마무리

여기까지 1) Go 에러의 특징, 2) API 서버에서 에러 핸들링을 어떻게 하면 좋을지, 3) 실제로 서버에서는 어떻게 적용했는지까지 살펴보았습니다.

- Go에서는 에러도 하나의 값이다

- 성공/실패를 명확하게 구분할 것

- 에러 래핑을 통해 추적 정보를 유지할 것

- 센티넬 에러, [errors.Is/As](http://errors.Is/As), ErrorHandler 등 다양한 기법을 상황에 맞게 활용하자

- 유지보수성과 가독성을 높이기 위해 일관된 에러 처리 방식을 갖추는 것이 중요하다

Go의 단순함과 함께 명료한 코드를 작성해보아요~!




**📚참고 자료**

Learning Go(2nd Edition), Jon Bodner, Chapter 9. Errors

[https://google.github.io/styleguide/go/decisions#handle-errors](https://google.github.io/styleguide/go/decisions#handle-errors)

[https://pkg.go.dev/errors](https://pkg.go.dev/errors)

[https://go.dev/blog/error-handling-and-go](https://go.dev/blog/error-handling-and-go)

[https://go.dev/blog/go1.13-errors](https://go.dev/blog/go1.13-errors)
