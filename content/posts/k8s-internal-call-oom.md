---
title: "ALB를 떼어냈더니 OOM이 났다 — Go HTTP 커넥션 풀 회고"
date: 2025-03-24T20:00:00+09:00
draft: false
tags: ["go", "kubernetes", "http", "networking", "backend"]
categories: ["backend"]
summary: "서비스 간 호출을 외부 도메인(ALB)에서 K8s 내부 서비스 네임으로 바꿨더니 OOM Kill이 발생했다. 커넥션 재사용이 사라지면서 유휴 TCP 연결이 누적된 사례와, Go HTTP 클라이언트의 커넥션 풀링·Keep-Alive·Idle Timeout을 다시 정리한 회고."
slug: "k8s-internal-call-oom"
ShowToc: true
TocOpen: false
---

## TL;DR

- 서비스 A → 서비스 B 호출 경로를 **외부 도메인(ALB)** 에서 **K8s 내부 서비스 네임(ClusterIP)** 으로 변경
- ALB가 빠지면서 매 요청마다 새로운 TCP 커넥션이 생성되고, 유휴 상태로 메모리에 누적
- 결국 서비스 B 컨테이너가 메모리 limit을 초과해 **OOM Kill** 발생
- 원인은 코드 변경이 아니라 **네트워크 계층에서 누가 커넥션을 관리하느냐**의 차이였다
- 해결은 결국 **HTTP 커넥션 풀링** + **Keep-Alive/Idle Timeout 튜닝**

## 무슨 일이 있었나

내부 서비스 두 개가 있다고 해봅시다.

- **서비스 A**: 클라이언트 측. 서비스 B의 HTTP API를 호출함.
- **서비스 B**: 백엔드 API. 서비스 A로부터 트래픽을 받음.

원래 A → B 호출은 외부 도메인을 거쳤습니다. K8s 클러스터 밖으로 한 번 나갔다가 ALB(Application Load Balancer)를 통해 다시 B로 들어오는 구조죠. "같은 클러스터 안에 있는데 굳이 ALB를 거쳐야 하나?" 라는 합리적인 의문에서, 호출 방식을 **K8s 내부 서비스 네임(ClusterIP)** 으로 바꾸는 변경이 적용되었습니다.

그리고 몇 시간 뒤, 서비스 B의 메모리 사용량이 limit을 초과하면서 **OOM Kill**이 발생했습니다. 코드는 바뀐 게 없는데 말이죠.

## 타임라인 (상대시간)

| 상대시간 | 내용 |
| --- | --- |
| T+0   | 서비스 B 신규 버전 배포 |
| T+3h  | 서비스 A 신규 버전 배포 (내부 도메인으로 호출 방식 변경) |
| T+6h  | 서비스 B Pod OOM Kill 발생 및 자동 재시작 |
| T+6h 13m | 서비스 A 일부 컨테이너만 외부 도메인 호출로 복구 |
| T+7h 10m | 서비스 B 롤백 |
| T+8h 10m | 서비스 A 호출을 외부 도메인 방식으로 완전 복구 |
| T+8h 25m | 메모리 사용량 감소 확인, 임시 복구 완료 |

가장 인상 깊었던 건 **B의 신규 배포 시점이 아니라 A의 호출 방식 변경 시점 이후부터 메모리가 새기 시작했다**는 점이었습니다. 그래서 처음에는 B의 새 버전을 의심했지만, 결국 원인은 A의 변경이었습니다.

## 왜 OOM이 났을까

원인은 **누가 커넥션을 관리하느냐**의 차이였습니다.

### 외부 도메인(ALB) 경유

```
서비스 A → ALB → 서비스 B
```

- ALB가 커넥션을 풀처럼 **관리하고 재사용**합니다.
- 클라이언트(A) 입장에서는 ALB와의 커넥션 하나로 여러 요청을 처리할 수 있고, ALB ↔ B 사이도 마찬가지로 재사용됩니다.
- 그래서 **B 입장에서 보이는 동시 커넥션 수가 안정적인 수준**으로 유지됩니다.

### K8s 내부 서비스 네임 경유

```
서비스 A → CoreDNS → ClusterIP → kube-proxy → 서비스 B Pod
```

- `kube-proxy`는 L4(Network Layer)에서 트래픽을 Pod로 라우팅하는 역할만 합니다.
- ALB처럼 **커넥션을 생성·유지·종료하거나 L7 로드 밸런싱을 해주지 않습니다.**
- 즉, 클라이언트(A)가 만든 TCP 커넥션이 그대로 B Pod로 흘러갑니다.
- 만약 클라이언트가 커넥션을 풀로 재사용하지 않으면, 매 요청마다 새 TCP 커넥션이 생성됩니다.

ALB가 해주던 "커넥션 재사용" 역할이 사라졌는데, A의 HTTP 클라이언트가 그걸 대신해줄 준비가 되어 있지 않았던 거죠. 매 요청마다 새 커넥션을 만들고, 그 커넥션이 **idle timeout(이 환경에서는 약 90초)** 동안 끊어지지 않고 메모리에 남으면서 누적되었습니다.

## 비교 테스트로 확인하기

가설을 검증하기 위해 동일한 부하 패턴(초당 5회, 10분, 총 약 3,000회 호출)으로 두 경로의 TCP 소켓 수를 비교했습니다.

| 경로 | 결과 |
| --- | --- |
| 내부 서비스 네임 호출 | 활성 TCP 소켓 수 **10 → 470 으로 급증**, ESTABLISHED 상태 유휴 연결 지속 누적 |
| ALB 경유 호출 | 소켓 수 **10개 내외로 안정 유지**, 유휴 연결 누적 없음 |

수치만 봐도 차이가 분명했습니다. 코드 한 줄 안 바꾸고 호출 경로만 바꿨을 뿐인데, 클라이언트 쪽에서 보이는 동시 커넥션 수가 한 자릿수에서 수백 단위로 뛰어버렸습니다.

## 그래서 어떻게 고쳤나

### 1) 임시 조치

- 서비스 B의 메모리 limit을 일시적으로 상향
- 서비스 A의 호출 경로를 외부 도메인(ALB) 방식으로 롤백

여기까지는 "원래대로 되돌렸다"에 가깝습니다. 진짜 해결책은 그다음입니다.

### 2) 근본 조치 — Go HTTP 클라이언트의 커넥션 풀링

Go의 `net/http` 패키지는 기본적으로 커넥션 풀(`http.Transport`)을 제공합니다. 문제는 **`http.Client`를 매 요청마다 새로 생성하면 풀의 이점을 전혀 못 누린다**는 점입니다.

```go
// ❌ 매 요청마다 새 Client/Transport — 커넥션 풀이 의미가 없음
func callB(ctx context.Context) error {
    client := &http.Client{Timeout: 3 * time.Second}
    resp, err := client.Get("http://service-b/api")
    // ...
}

// ✅ 패키지 전역 또는 컴포넌트 단위로 단일 Client 재사용
var httpClient = &http.Client{
    Timeout: 3 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
    },
}

func callB(ctx context.Context) error {
    resp, err := httpClient.Get("http://service-b/api")
    // ...
}
```

핵심 포인트:

- **`http.Client`(정확히는 그 안의 `Transport`)를 재사용**해야 커넥션 풀이 동작합니다.
- `MaxIdleConnsPerHost`는 호스트별 유휴 커넥션 상한입니다. 기본값(2)은 트래픽이 어느정도 있는 서비스 간 호출에는 보통 부족합니다.
- `IdleConnTimeout`이 지나면 풀 안의 유휴 커넥션이 정리됩니다.

### 3) 운영 측 조치

- **모니터링 강화**: 커넥션 수(Pod별 ESTABLISHED 소켓 수)와 네트워크 트래픽 지표를 대시보드에 추가
- **변경 시 영향 범위 점검 체크리스트**: "호출 경로/네트워크 계층이 바뀌는 변경"은 코드 변경이 아니더라도 부하 테스트 대상

## 회고 — 무엇을 배웠나

1. **"같은 클러스터 안이니까 ALB는 불필요한 홉이다"는 단순한 직관**이 틀릴 수 있다.
   ALB는 단순한 트래픽 라우터가 아니라 **커넥션 관리자** 역할도 겸하고 있었다. 그 책임이 어디로 옮겨가는지를 같이 봐야 한다.

2. **OOM Kill의 원인이 항상 그 컨테이너 코드에 있는 건 아니다.**
   이번 사례에서 메모리가 새던 곳은 B였지만, 원인은 A의 호출 방식 변경이었다. **누가 메모리를 쓰는가**와 **누가 트리거하는가**는 다를 수 있다.

3. **HTTP 클라이언트는 재사용해야 한다.**
   Go에서 `http.Client`를 함수 안에서 매번 새로 만드는 코드는 의외로 흔하다. 호출 빈도가 낮으면 티가 안 나다가, 트래픽이 늘거나 네트워크 계층이 바뀌면 한 번에 터진다.

4. **K8s ClusterIP 호출은 L4 라우팅이다.**
   `kube-proxy`는 L7 LB가 아니다. 커넥션 재사용·Keep-Alive·로드 밸런싱 알고리즘 같은 건 클라이언트(또는 사이드카, 서비스 메시)의 책임으로 내려온다.

## 참고

- [Go `net/http` 패키지](https://pkg.go.dev/net/http)
- [Go `http.Transport`](https://pkg.go.dev/net/http#Transport)
- [Kubernetes Services와 kube-proxy 동작 방식](https://kubernetes.io/docs/concepts/services-networking/service/)
