---
title: "GC 많이 된다, 스트레스 많이 받을거야 — validator.New() 싱글톤 전환으로 GC 압력 줄이기"
date: 2026-03-24T20:00:00+09:00
draft: false
tags: ["go", "validator", "gc", "performance", "fx", "backend"]
categories: ["go"]
summary: "Go의 go-playground/validator 인스턴스를 요청마다 New()로 만들고 있던 코드를 fx DI 싱글톤으로 바꿨다. 단건 처리는 25배 빨라지고 메모리 할당은 96배 줄었지만, 진짜 효과는 GC 압력 감소로 p99 스파이크가 잡힌다는 점이었다. 벤치마크와 gctrace로 확인한 GC 거동 차이까지 정리."
slug: "go-validator-singleton-gc"
ShowToc: true
TocOpen: false
---

안녕하세요, 레인입니다 ☔️

![validator 싱글톤 전환으로 GC 압력 줄이기](/images/go-validator-singleton-gc/d35ce639-32b1-479e-a7eb-7358eedafd5c.png)

오늘은 한 줄짜리 코드 변경이 GC 거동에 어떤 영향을 주는지 들여다본 이야기입니다.

> `validator.New()` 한 번만 만들어 재사용하기

너무 당연해 보이는 결론이지만, "왜 그래야 하는가"를 벤치마크와 `GODEBUG=gctrace=1`로 끝까지 확인해보면 GC 압력이라는 흥미로운 면이 드러납니다.

## 1. 무엇이 문제였나

API 요청 검증에 [`go-playground/validator`](https://github.com/go-playground/validator)를 쓰고 있었습니다. 그런데 컨트롤러마다 이렇게 매 요청마다 인스턴스를 새로 만들고 있었어요.

```go
// Before: 매 요청마다 새 인스턴스 생성
func (c *UserController) CreateUser(ctx *fiber.Ctx) error {
    if err := validator.New().Struct(&req); err != nil {
        // ...
    }
}
```

`validator.New()`는 가벼운 함수가 아닙니다. 내부적으로 다음과 같은 초기화를 수행합니다.

- Go 내장 타입별 검증 함수 등록 (string, int, float, slice, map 등)
- `required`, `min`, `max`, `email` 등 빌트인 태그 함수 맵 초기화
- reflect 기반 struct 캐시 구조체 생성
- 태그 파서, 에러 풀 등 내부 자료구조 할당

이게 **요청마다 반복**되고 있었습니다. CPU 시간도 시간이지만, 더 신경 쓰이는 건 **그때마다 단기 객체가 힙에 쌓인다**는 점이었어요.

## 2. 해법은 너무 단순하다 — 싱글톤

`*validator.Validate`는 thread-safe합니다 ([공식 문서](https://pkg.go.dev/github.com/go-playground/validator/v10#Validate)). 즉 **하나만 만들어 모든 핸들러에서 공유**해도 안전합니다.

저는 [fx](https://github.com/uber-go/fx) DI 컨테이너를 쓰고 있어서, provider로 한 번 등록해두면 알아서 싱글톤으로 관리됩니다.

```go
// After: fx DI를 통한 싱글톤 주입

// --- provider 등록 (module.go 등) ---
var Module = fx.Provide(
    validator.New,         // fx가 싱글톤으로 관리
    NewUserController,
)

// --- controller 구조체 ---
type UserController struct {
    validate *validator.Validate
}

func NewUserController(validate *validator.Validate) *UserController {
    return &UserController{validate: validate}
}

// --- 핸들러 ---
func (c *UserController) CreateUser(ctx *fiber.Ctx) error {
    if err := c.validate.Struct(&req); err != nil {
        // ...
    }
}
```

fx를 쓰지 않더라도 `var defaultValidator = validator.New()` 같은 패키지 레벨 변수로 풀어도 동일합니다. 핵심은 **인스턴스를 한 번만 만든다**는 것.

## 3. 정말 빨라지는지 — 벤치마크

말로만 "재사용이 좋다"라고 하긴 좀 그래서, Go 벤치마크로 직접 비교했습니다.

```go
// 싱글톤 validator 재사용
func BenchmarkValidatorSingleton(b *testing.B) {
    validate := validator.New()
    req := makeBenchRequest()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = validate.Struct(req)
    }
}

// 매번 새로운 validator 생성
func BenchmarkValidatorNew(b *testing.B) {
    req := makeBenchRequest()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        v := validator.New()
        _ = v.Struct(req)
    }
}
```

결과:

```
go test -bench=BenchmarkValidator -benchmem ./...

BenchmarkValidatorSingleton-11   1,999,886       585 ns/op       216 B/op     10 allocs/op
BenchmarkValidatorNew-11            82,887    14,530 ns/op    20,703 B/op    292 allocs/op
```

| 지표 | 매번 `New()` | 싱글톤 | 개선 배율 |
| --- | --- | --- | --- |
| 1회 소요 시간 | 14,530 ns | 585 ns | **25배 빠름** |
| 1회당 힙 메모리 할당량 | 20,703 B | 216 B | **96배 절감** |
| 1회당 힙 할당 횟수 | 292회 | 10회 | **29배 절감** |
| 같은 시간 총 반복 횟수 | 82,887 | 1,999,886 | **24배 많음** |

깔끔하게 차이가 납니다.

### 그런데 사실, 체감 차이는 미미합니다

여기서 솔직해질 필요가 있어요. 단건 기준으로 보면 이 수치들은 작습니다.

```
14,000 ns = 0.014 ms   ← DB 쿼리 한 번(1~10ms)에 비하면 미미
20,000 B  = 0.02 MB    ← 단일 요청 기준으로는 무시할 수준
```

요청 하나의 응답시간이 0.014ms 빨라진다고 사용자가 체감하지 않습니다. 메모리 20KB가 더 잡힌다고 OOM이 나지도 않고요. validator가 전체 요청 처리에서 차지하는 비중이 그만큼 작기 때문입니다.

**그럼 왜 굳이?** 누적되면 이야기가 달라지기 때문입니다.

## 4. 누적되면 달라지는 이야기 — GC 압력

예시 시나리오로 **초당 200 요청** (분당 12,000)이 들어오는 서비스를 가정해봅시다. 매번 `New()`로 만들면:

| 지표 | 매번 `New()` | 싱글톤 | 효과 |
| --- | --- | --- | --- |
| 메모리 할당/초 | ~4.14 MB | ~0.04 MB | **~4.10 MB 절감** |
| 힙 할당 횟수/초 | 58,400회 | 2,000회 | 56,400회 절감 |

초당 4.14MB의 **단기 객체**가 힙에 쌓입니다. 이 객체들은 요청이 끝나면 바로 쓸모없어지지만, **GC가 돌기 전까지는 live 힙에 포함**되어 있죠.

Go GC는 `GOGC` 비율(기본 100%, 즉 live 힙의 2배)에 도달하면 트리거됩니다. 단기 객체가 빠르게 쌓이면 그만큼 자주 트리거되고, 매번 STW(Stop-The-World) 마킹/스윕이 끼어듭니다. 이게 **p99 레이턴시 스파이크의 흔한 원인** 중 하나입니다.

평균 응답 시간은 멀쩡한데 가끔 한 번씩 튀는 요청이 있다? GC가 그 자리에 들어가 있을 가능성이 큽니다.

### `GODEBUG=gctrace=1`로 확인

가설을 확인하려고 두 벤치마크를 `gctrace` 켜고 돌렸습니다.

```
GODEBUG=gctrace=1 go test -bench=BenchmarkValidator -benchmem ./...
```

요약하면 이렇게 나왔어요.

| 항목 | 싱글톤 | `validator.New()` 매번 |
| --- | --- | --- |
| GC 발생 횟수 | 274회 | 346회 (1.3배) |
| GC CPU 사용률 | 0% | 1~2% |
| Live 힙 | **1 MB — 완전히 평탄** | **5~8 MB — 톱니 패턴** |
| GC Goal | 4 MB 고정 (Go 최소값) | 11~16 MB 변동 |
| Concurrent Marking | 0.15~0.25 ms | 0.50~0.95 ms (3~4배) |

읽고 나면 그림이 보입니다.

- **싱글톤**: 단기 객체가 거의 생성되지 않으니 live 힙이 1MB 부근에서 평평합니다. GC는 사실상 유휴 상태에 가까워요.
- **매번 New()**: `validator.New()`가 한 번에 292회의 힙 할당을 만들어내고, 그 단기 객체들이 쌓였다 GC에 쓸려나가는 과정이 반복됩니다. 그래서 **live 힙 그래프가 톱니 모양**이 됩니다.

특히 **Concurrent Marking 시간이 3~4배 늘어난다**는 게 눈에 띕니다. 마킹은 STW가 아니라 동시에 돌긴 하지만, 그동안 mutator(애플리케이션 코드)와 CPU를 같이 쓰니까 결국 throughput과 tail latency 모두에 영향을 줍니다.

## 5. 결론 — 한 줄짜리 변경이 한 일

요약하면 이렇습니다.

- `validator.New()`를 요청마다 호출하던 코드를 fx DI가 관리하는 싱글톤으로 바꿨다.
- 단건 처리 속도 **25배**, 메모리 할당 **96배** 개선이 벤치마크로 확인되었다.
- 하지만 진짜 효과는 **GC 압력 감소**다. 단기 객체 폭증이 사라지면서 live 힙이 평탄해지고, GC 호출 빈도와 마킹 시간이 줄어든다.
- 그 결과 가끔 튀던 **p99 스파이크가 완화**된다. 평균 응답 시간은 멀쩡한데 가끔 느린 요청이 있는 서비스라면 한 번 의심해볼 만한 패턴이다.

### 일반화 — 다른 곳에도 적용 가능한 패턴

`validator`만의 이야기는 아닙니다. **"초기화가 무겁고 thread-safe하지만 매번 새로 만들고 있는 것"** 들이 의외로 코드베이스 곳곳에 숨어있어요.

대표 후보:

- `*regexp.Regexp` — `regexp.MustCompile`은 비싸고, 컴파일된 Regexp는 thread-safe. **함수 안에서 매번 컴파일하는 코드 흔하게 보임.**
- `*http.Client` — 새로 만들면 커넥션 풀이 의미가 없음 (이건 [지난 글](/posts/k8s-internal-call-oom/)에서 다뤘죠).
- `*tls.Config`, `*sql.DB`, `*template.Template` — 비슷한 결.

**"매번 만들 필요가 있는가?"** 라는 질문을 한 번씩 던져보면 의외의 GC 절약 포인트가 보입니다.

## 참고

- [go-playground/validator 공식 문서](https://pkg.go.dev/github.com/go-playground/validator/v10)
- [Uber fx](https://github.com/uber-go/fx)
- [Go GC 가이드](https://tip.golang.org/doc/gc-guide)
- [`GODEBUG=gctrace=1` 출력 해석](https://pkg.go.dev/runtime#hdr-Environment_Variables)
