---
title: "Go는 개발자를 믿어… 그래서 panic을 찢어 🤯"
date: 2025-04-11T18:00:00+09:00
draft: false
tags: ["go", "panic", "recover", "backend"]
categories: ["go"]
summary: "Go의 panic은 단순한 종료 신호가 아니라 고루틴 단위로 전파되는 치명 신호다. panic이 무엇이고 왜 발생하는지, 프로덕션 웹 서버에서 recover 체인을 어떻게 구성해야 하는지 정리."
slug: "golang-panic-recover"
ShowToc: true
TocOpen: false
cover:
  image: "/images/golang-panic-recover/golang-panic-recover.png"
  alt: "Go panic 전파 흐름"
  relative: false
  hidden: true
---

# Quiz!

Go에서 panic이 발생하면 어떤 일이 발생할까요? 대답해보아요 🎶

1. 프로그램이 종료된다
2. 고루틴이 종료된다
3. 잘 모르겠다
4. 이외의 대답

….

1번으로 대답하신 분들 주목!!!! 그렇게 생각하신다면 경기도 오산(…)입니다.

[go 문서](https://pkg.go.dev/builtin#panic)에서 panic에 대한 설명은 다음과 같습니다.

> The panic built-in function stops normal execution of the current goroutine.
> When a function F calls panic, normal execution of F stops immediately.
> Any functions whose execution was deferred by F are run in the usual way, and then F returns to its caller.
> To the caller G, the invocation of F then behaves like a call to panic, terminating G's execution and running any deferred functions.
> This continues until all functions in the executing goroutine have stopped, in reverse order.
>
> At that point, the program is terminated with a non-zero exit code. This termination sequence is called panicking and can be controlled by the built-in function recover.
>
> \[요약\]
>
> **panic은 현재 고루틴의 실행을 중단시키는** 내장함수입니다. 해당 함수에서 defer된 함수를 모두 실행한 다음, 상위 호출자(caller)에게 패닉을 전달합니다. 패닉이 상위 함수로 전파되는 것을 패닉킹(panicking)이라고 하며, 이것은 recover()을 통해 제어할 수 있습니다.

# 패닉은 왜 발생시킬까요?

런타임 도중에 호출자에게 '나 지금 위험에 빠졌어' 라는 신호를 보낼때, 에러 대신 패닉을 호출하는 이유가 뭘까요? 에러로도 충분히 알릴 수 있는데 말이죠. **복구 불가능하거나 심각한 논리 오류 상황에서** 패닉을 호출합니다.

### 패닉 상황은 이렇게 분류할 수 있습니다.

1. **의도된 패닉**
    1. 프로그램의 초기화 시점
    2. 프로그램 진행에 치명적인 위험을 줄 수 있는 부분을 일부러 패닉 처리

2. **의도되지 않은 패닉**
    1. 논리적 오류
        1. 슬라이스 인덱스 범위 초과
        2. 타입 단언 실패
        3. invalid memory address or nil pointer dereference

    2. 시스템 수준 문제
        1. 스택 오버플로우
        2. go 런타임 내부 문제

### 왜 이런 사소한 실수도 panic일까요?

의도치 않게 발생하는 패닉은 그닥 치명적으로 보이지 않을 수도 있습니다. 슬라이스 인덱스를 잘못 쓰거나, nil을 참조하는 실수 정도인데, 왜 go는 이런 상황을 panic으로 처리할까요?

그 이유는 바로 **Go가 개발자를 믿는 언어이기 때문**입니다. Go는 개발자가 기본적인 논리 오류는 사전에 방지할 수 있다고 가정합니다. 저런 에러들은 명백한 프로그래밍 실수이고, 즉시 고쳐야 함을 알리는 신호입니다.

그렇지만 모든 개발자가 믿음직한 것은 아니니 실수를 하기도 합니다. 예상치 못한 시점에서 패닉킹이 일어나 프로그램이 종료되는 불상사가 일어나게 됩니다. 😩

### 의도되지 않은 패닉이 프로덕션 웹 서버에서 발생하면?

예상치 못한 panic이 recover되지 않으면 해당 고루틴의 호출 스택을 따라 전파되고, 최종적으로 main에 도달하면 전체 프로그램이 종료됩니다.

즉, 하나의 요청 처리 중 발생한 panic이 서버 전체를 강제 종료 시키는 결과를 초래합니다.

![panic 전파 흐름](/images/golang-panic-recover/golang-panic-recover.png)

**문제가 발생한 고루틴만 종료시키면 되지 않을까요?** 이를 방지하지 위해 recover 체인이 필요합니다.

### Go는 기본적으로 recover를 제공할까?

이런 문제를 방지하기 위해 net/http에서는 기본적으로 recover 로직이 내장되어 있으며(아래 예시 코드 참고), go의 웹 프레임워크인 gin, echo, fiber에서는 recovery 미들웨어를 제공하고 있습니다. 의도치 않은 panic에 대비해 서버 인터셉터 혹은 미들웨어로 recovery 체인을 추가하는게 권장되는 패턴이기도 합니다.

```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    var p *string
    fmt.Println(*p) // panic: nil pointer dereference
}

func main() {
    http.HandleFunc("/hello", handler)
    err := http.ListenAndServe(":9000", nil)
    if err != nil {
        panic(err)
    }
}
```

위 서버를 동작시키고 /hello api를 호출해보면, api에서 panic이 발생함에도 불구하고, 프로그램이 종료되지 않습니다. **net/http에서 recover가 내장되어 있기 때문**입니다.

fiber에서는 recovery 체인을 다음과 같이 추가합니다.

```go
// Initialize default config
app.Use(recover.New())

// This panic will be caught by the middleware
app.Get("/", func(c *fiber.Ctx) error {
    panic("I'm an error")
})
```

### panic을 감추는 것이 아니라, 드러내기 위함입니다

recover()를 사용하는 이유는 panic을 숨기기 위함이 아닙니다. 오히려 **panic을 명확히 드러내고, 그에 맞게 대처할 수 있도록 하기 위한 수단**입니다.

**실패한 요청은 실패하게 두되, 다른 요청에 영향을 주지 않는 것**이 핵심입니다.

### 현실적인 설계 방법

- **초기화 시 panic** → 그대로 종료 (의도된 중단)
- **요청 처리 중 panic** → recover로 해당 요청만 실패
- **복구 불가능한 내부 손상** → 강제 종료

# 결론

panic은 단순히 프로그램을 죽이기 위한 수단이 아닙니다. 치명적인 상태에 빠졌다는 신호이자, 개발자에게 보내는 경고입니다.

그러나 의도치 않은 panic이 발생했을 때 전체 프로그램이 중단되지 않도록 recover 체인을 적절히 구성하고, 로그를 남기고, 고루틴을 격리하는 것은 Go 웹 서버를 운영하는 데 있어 필요한 패턴입니다.

피드백은 언제든 환영합니다 🤗

**📚 참고**

- [https://pkg.go.dev/builtin#panic](https://pkg.go.dev/builtin#panic)
- [https://go.dev/doc/effective_go#panic](https://go.dev/doc/effective_go#panic)
- [https://go.dev/blog/defer-panic-and-recover](https://go.dev/blog/defer-panic-and-recover)
