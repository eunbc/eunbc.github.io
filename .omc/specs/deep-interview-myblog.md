# Deep Interview Spec: 개인 포트폴리오 블로그 (Hugo PaperMod)

## Metadata
- Interview ID: myblog-2026-05-09
- Rounds: 4
- Final Ambiguity Score: 10.5%
- Type: greenfield
- Generated: 2026-05-09
- Threshold: 20%
- Status: PASSED

## Clarity Breakdown
| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Goal Clarity | 0.85 | 0.40 | 0.34 |
| Constraint Clarity | 0.95 | 0.30 | 0.285 |
| Success Criteria | 0.90 | 0.30 | 0.27 |
| **Total Clarity** | | | **0.895** |
| **Ambiguity** | | | **0.105** |

## Goal
티스토리에서 벗어나 **이직/취업용 포트폴리오 + 개인 아카이브** 두 역할을 동시에 수행하는 커스텀 개인 블로그를 구축한다. Confluence에 쌓아둔 자랑스러운 글 중 10개 이하를 선별·이전해 첫 콘텐츠로 채우고, 사이트 자체는 Hugo PaperMod 테마를 가볍게 커스터마이징(색상·폰트·레이아웃 미세 조정)한다. 채용담당자가 둘러보며 "정리력과 결과물이 보인다"고 느끼게 만드는 것이 1순위, 본인이 자기 글을 모아두고 보면서 만족감을 얻는 것이 2순위.

## Constraints
- **디자인 스코프**: 멋진 테마(PaperMod) + 색상/폰트/약간의 컴포넌트 튜닝까지. 처음부터 짜지 않음.
- **기술 스택**: Hugo (Go 기반 SSG, GoLand 사용자에게 친숙)
- **테마**: PaperMod (이미 세련됨, 다크모드/검색/태그/RSS 내장)
- **배포**: GitHub Pages
- **도메인**: `<username>.github.io` 서브도메인 (커스텀 도메인 구매 안 함, 비용 0원)
- **콘텐츠 마이그레이션**: Confluence → Markdown **수기 복붙** (10개 이하, 텍스트 위주)
- **시간 예산**: 주말 단위 (몇 시간~하루)
- **유지비**: 0원 / 자동 빌드는 GitHub Actions
- **콘텐츠 큐레이션**: 자랑스러운 글만 골라서 — 양보다 질

## Non-Goals
- 처음부터 모든 컴포넌트를 직접 디자인하는 풀커스텀 작업
- 50개 이상 대량 마이그레이션 또는 Confluence REST API 자동화 파이프라인
- 커스텀 도메인 구매·연결
- 댓글 시스템(Disqus, giscus) — 1차 출시에서는 제외 (필요 시 v2)
- SEO/SNS 그로스 최적화 — 검색 유입이 1순위가 아님
- Astro/Next.js 같은 React 기반 프레임워크 (현재 디자인 스코프 대비 오버킬)
- 데이터베이스, 인증, 동적 백엔드

## Acceptance Criteria
- [ ] `<username>.github.io` URL에 사이트가 떠 있고 외부에서 접근 가능
- [ ] PaperMod 테마가 적용되어 있고, 색상/폰트는 본인 취향으로 1회 이상 커스터마이징됨
- [ ] About 페이지에 본인 소개·연락처·GitHub 링크가 있음
- [ ] Confluence에서 가져온 글 1개 이상이 마크다운으로 게시되어 있음 (목표 10개 이하)
- [ ] 글 목록 페이지에서 게시글이 시간순으로 정렬되어 노출됨
- [ ] 다크모드 토글, 태그 필터, 검색 기능 동작
- [ ] `git push` 시 GitHub Actions로 자동 빌드/배포가 동작 (수동 빌드 불필요)
- [ ] 모바일 반응형으로 깨지는 곳 없음
- [ ] 채용담당자가 30초 안에 "이 사람이 뭘 했는지" 파악할 수 있는 first impression

## Assumptions Exposed & Resolved
| Assumption | Challenge | Resolution |
|------------|-----------|------------|
| "커스텀으로 만들고 싶다" = 처음부터 짜겠다 | 디자인 스코프 직접 질문 | 테마 + 색상/폰트만 손봄. "예쁨"은 테마 선택으로 달성. |
| 포트폴리오 vs 블로그 둘 중 하나여야 함 | 1순위 목적 질문 | 둘 다 (이직용 1순위 + 아카이브 만족감 2순위). 같은 사이트가 양쪽을 모두 만족. |
| 컨플루언스 글 전체를 옮겨야 함 | 마이그레이션 규모 질문 | 자랑스러운 10개 이하만. 큐레이션이 곧 포트폴리오 시그널. |
| GitHub Pages면 무조건 Jekyll | 스택 옵션 제시 | Hugo PaperMod로 결정. Go 친숙 + 빌드 빠름 + 테마 세련됨. |

## Technical Context
- **현재 디렉토리 상태**: `/Users/eunbi/GolandProjects/myblog` — 빈 디렉토리 (`.omc/`만 존재), Git 미초기화
- **사용자 환경**: macOS (Darwin), GoLand IDE 사용, Go 생태계 친숙
- **필요한 사전 작업**:
  1. Hugo 설치 (`brew install hugo`)
  2. GitHub repo 생성 (`<username>.github.io` 형태로)
  3. Hugo 사이트 초기화 + PaperMod submodule 추가
  4. GitHub Actions 워크플로(`hugo.yml`) 작성
- **알려진 미정 변수 (실행 단계에서 확정)**:
  - GitHub username (repo 이름 결정)
  - 메인 색상/폰트 선호 (PaperMod 커스터마이징)
  - 첫 게시할 Confluence 글 1~3개 후보

## Ontology (Key Entities)
| Entity | Type | Fields | Relationships |
|--------|------|--------|---------------|
| Blog Site | core | url, theme, primary_color, font | hosts Posts, has Pages |
| Post | core | title, date, tags, markdown_body, source_confluence_url | belongs to Blog Site |
| Page (About/Projects) | core | slug, content | belongs to Blog Site |
| Confluence Source | external | original_url, exported_text | source of Post |
| Recruiter / Reader | actor | — | views Blog Site |

## Ontology Convergence
| Round | Entity Count | New | Changed | Stable | Stability Ratio |
|-------|-------------|-----|---------|--------|----------------|
| 1 | 5 | 5 | - | - | N/A |
| 2 | 5 | 0 | 0 | 5 | 100% |
| 3 | 5 | 0 | 0 | 5 | 100% |
| 4 | 5 | 0 | 0 | 5 | 100% |

수렴 완료: 4라운드 연속 동일 엔티티 — 도메인 모델 안정.

## Interview Transcript
<details>
<summary>Full Q&A (4 rounds)</summary>

### Round 1 — Goal Clarity
**Q:** 이 블로그의 1순위 목적이 뭐예요?
**A:** 1번(이직용 포트폴리오) + 3번(개인 아카이브) 둘 다
**Ambiguity:** 62% (Goal: 0.65, Constraints: 0.20, Criteria: 0.20)

### Round 2 — Constraint Clarity
**Q:** 디자인을 어느 레벨까지 다루고 싶으세요?
**A:** 멋진 테마 고르고 색상·폰트만 살짝
**Ambiguity:** 51% (Goal: 0.65, Constraints: 0.55, Criteria: 0.20)

### Round 3 — Success Criteria
**Q:** Confluence에서 옮겨올 글이 대략 어떤 규모/형태예요?
**A:** 10개 이하, 텍스트 위주 — 수기 복붙
**Ambiguity:** 22.5% (Goal: 0.85, Constraints: 0.80, Criteria: 0.65)

### Round 4 — Success Criteria (final)
**Q:** 기술 스택 + 배포 조합을 고르면 '완성'의 정의가 완성돼요. 어떤 조합?
**A:** Hugo (PaperMod) + GitHub Pages + .github.io 서브도메인
**Ambiguity:** 10.5% ✅

</details>
