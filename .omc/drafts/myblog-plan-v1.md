# Plan v1 — Hugo PaperMod 개인 포트폴리오 블로그

> Source spec: `.omc/specs/deep-interview-myblog.md` (ambiguity 10.5%, PASSED)

## RALPLAN-DR Summary (Short Mode)

### Principles
1. **테마 신뢰**: PaperMod의 검증된 디자인을 기반으로 시작하고, 색상·폰트만 살짝 손본다. 풀커스텀 금지.
2. **콘텐츠 큐레이션 = 포트폴리오**: 양보다 질. 자랑스러운 글 5–10개로 시작하고 점진 추가.
3. **0원 운영**: 도메인·호스팅·유지비 모두 무료. GitHub Pages + 서브도메인.
4. **자동 빌드**: GitHub Actions로 push → 자동 배포. 수동 빌드 단계 없음.
5. **점진적 확장**: 1차에서는 댓글·SEO·커스텀 도메인 제외. 필요 시 v2.

### Decision Drivers (Top 3)
1. **시간 예산 = 주말** → 가장 빠르게 배포 가능한 경로 우선
2. **이직용 first impression** → 30초 내 "이 사람 정리력 있음"이 보여야 함
3. **유지비/관리 부담 0** → 빌드 자동화, 의존성 최소화

### Viable Options Considered
- **Option A (선택)**: Hugo + PaperMod + GitHub Pages + GitHub Actions
  - Pros: 빌드 초고속(<1초), Go 친숙, PaperMod가 이미 세련됨, GitHub Actions로 zero-config 배포, 무료
  - Cons: 디자인 변경 시 Go 템플릿 문법 학습 필요(가벼운 수정엔 불필요)
- **Option B**: Hugo + PaperMod + Cloudflare Pages
  - Pros: Cloudflare CDN, preview deploys, 더 빠른 글로벌 응답
  - Cons: 별도 계정/연결 단계, 본인이 GitHub Pages를 명시 선택함 → **invalidated**
- **Option C**: Hugo `hugo deploy` + AWS S3
  - Pros: 완전한 제어
  - Cons: AWS 계정/요금/IAM 복잡도, "0원" 원칙 위반 → **invalidated**

선택 근거: 사용자가 GitHub Pages를 명시 선택했고, 모든 5개 원칙(특히 "0원 운영"·"자동 빌드")을 만족하는 유일한 옵션.

---

## Requirements Summary
- 빈 디렉토리 `/Users/eunbi/GolandProjects/myblog`에 Hugo 기반 정적 사이트를 부트스트랩
- PaperMod 테마 적용 (Git submodule)
- 색상/폰트 1회 이상 커스터마이징 (사용자 입력 받음)
- About 페이지에 본인 소개·연락처·GitHub 링크
- Confluence에서 옮긴 마크다운 글 1개 이상 게시 가능 상태
- GitHub Actions 워크플로(`hugo.yml`)로 push → 자동 배포
- `<username>.github.io` URL에서 외부 접근 가능
- 다크모드/태그/검색 기능 동작 (PaperMod 기본 제공)

## Acceptance Criteria (testable)
- [ ] `hugo version`이 v0.120 이상 출력
- [ ] `git remote -v`에 `<username>/<username>.github.io.git` 등록됨
- [ ] `themes/PaperMod/` submodule 존재 (`.gitmodules` 확인)
- [ ] `hugo --gc --minify` 명령이 에러 없이 `public/` 생성
- [ ] `public/index.html`이 PaperMod 마크업을 포함 (예: `class="entry"`)
- [ ] `hugo.toml` 또는 `config.yaml`에 `theme = "PaperMod"`, `baseURL = "https://<username>.github.io/"` 명시
- [ ] `content/posts/` 하위에 마크다운 글 1개 이상 존재
- [ ] About 페이지(`content/about.md` 또는 `content/about/_index.md`) 존재 + GitHub 링크 포함
- [ ] `assets/css/extended/custom.css` (또는 동등 파일)에 색상·폰트 오버라이드 1개 이상
- [ ] `.github/workflows/hugo.yml` 존재, `actions/configure-pages` + `actions/deploy-pages` 사용
- [ ] GitHub repo Settings → Pages → Source = `GitHub Actions`로 설정 안내됨 (체크리스트로 사용자에게 위임)
- [ ] push 후 Actions 탭에서 워크플로 성공 확인 (사용자 검증 단계)
- [ ] 배포된 `https://<username>.github.io/`에서 메인 + 글 1개 진입 가능
- [ ] 모바일 너비(≤375px)에서 가로 스크롤 없음 (사용자 시각 확인)

## Implementation Steps

### Phase A — 사용자 입력 수집 (Autopilot Phase 2 시작 시)
1. 사용자에게 3가지 placeholder 값을 묻는다 (`AskUserQuestion`):
   - GitHub username
   - 메인 색상 (hex 또는 톤: 예 "warm beige", "deep navy")
   - 폰트 선호 (예: "Pretendard", "Noto Sans KR + JetBrains Mono")
2. 선택적으로 첫 게시 글 1개의 마크다운 파일/텍스트를 받음 (없으면 스켈레톤 글 1개 생성)

### Phase B — 로컬 부트스트랩
3. Hugo 설치 확인: `hugo version`. 없으면 `brew install hugo` 안내 (사용자 위임)
4. `hugo new site . --force` (현재 디렉토리에 부트스트랩, `.omc/`는 Hugo가 무시)
5. `git init` + `.gitignore`에 `public/`, `resources/`, `.hugo_build.lock` 추가
6. PaperMod submodule 추가:
   ```bash
   git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
   ```
7. `hugo.toml` 작성:
   - `baseURL = "https://<username>.github.io/"`
   - `languageCode = "ko-kr"`, `defaultContentLanguage = "ko"`
   - `theme = ["PaperMod"]`
   - `[params]` 블록: `defaultTheme = "auto"`, `ShowReadingTime = true`, `ShowShareButtons = false` (1차 SNS 제외 원칙), `ShowToc = true`
   - `[outputs]` `home = ["HTML", "RSS", "JSON"]` (PaperMod 검색 기능 활성화)
   - `[menu.main]` 항목: Home, Posts, About, Tags
   - `[params.profileMode]` 또는 `[params.homeInfoParams]` 중 하나로 메인 hero 구성

### Phase C — 콘텐츠 스캐폴드
8. About 페이지: `content/about.md` 생성 (front matter `title: "About"`, `layout: "page"`, `url: "/about/"`). 본문에 본인 이름, 한 줄 소개, GitHub 링크, 이메일.
9. 첫 게시글: `content/posts/<slug>.md` (사용자 제공 텍스트 사용 또는 placeholder 글 1개)
10. (선택) `content/projects/_index.md`로 포트폴리오 섹션 (1차에서는 single section 운영, 추후 분리 가능)

### Phase D — 디자인 튜닝
11. `assets/css/extended/custom.css` 생성:
    - `:root { --primary: <hex>; --secondary: <hex>; }` 오버라이드
    - `body { font-family: <chosen>; }`
    - 필요 시 `--theme: "<chosen-theme-light>"`, `--entry: "<chosen-theme-light>"`
12. 폰트가 웹폰트면 `layouts/partials/extend_head.html`에서 `<link>` 추가 (PaperMod의 partial extension 메커니즘)
13. 로컬 검증: `hugo server -D` → http://localhost:1313 확인

### Phase E — 배포 파이프라인
14. GitHub repo 생성 (사용자 단계 위임): repo 이름 = `<username>.github.io`
15. `.github/workflows/hugo.yml` 작성 (Hugo 공식 GitHub Pages 워크플로 템플릿 기반):
    - trigger: `push` to `main`
    - jobs:
      - `build`: `actions/checkout@v4` (with `submodules: recursive`), Hugo install (`peaceiris/actions-hugo@v2` 또는 공식 install step), `hugo --gc --minify --baseURL ${{ steps.pages.outputs.base_url }}/`, `actions/upload-pages-artifact@v3`
      - `deploy`: `actions/deploy-pages@v4`, permissions `pages: write`, `id-token: write`
16. 사용자 단계 안내(체크리스트로 노출):
    - GitHub repo Settings → Pages → Source = `GitHub Actions`
    - `git remote add origin git@github.com:<username>/<username>.github.io.git`
    - `git push -u origin main`
17. Actions 탭에서 빌드 성공 확인 후 `https://<username>.github.io/` 접근 검증

### Phase F — 검증 & 마이그레이션 가이드
18. README.md 작성 (간단): 로컬 실행, 글 추가, 빌드/배포 흐름
19. Confluence → 마크다운 마이그레이션 가이드를 README 또는 `docs/migration.md`에 명시:
    - Confluence 페이지에서 "..." 메뉴 → Export → Word/HTML 후 수기 마크다운 변환
    - 또는 페이지 본문 복사 → 마크다운 에디터(예: VSCode + Markdown All in One)에서 정리
    - front matter 템플릿 제공 (`title`, `date`, `tags`, `draft`, `summary`)
20. 최종 체크: 위 Acceptance Criteria 13개 항목 검증

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Hugo 미설치 / 버전 불일치 | M | M | Phase B 첫 단계에서 `hugo version` 확인, 없으면 brew install 안내 |
| GitHub repo 이름 컨벤션 실수 (`<username>.github.io` 아닌 다른 이름) | M | H | 명시적 안내 + `baseURL`을 placeholder로 처리, push 전 체크리스트 |
| PaperMod submodule이 `git clone` 시 누락 | H | H | `actions/checkout`에 `submodules: recursive` 명시, 로컬도 `git clone --recurse-submodules` 안내 |
| GitHub Pages Source 설정을 "GitHub Actions"로 안 바꿔서 404 | H | H | 체크리스트로 명시하고, 첫 push 후 사용자 확인 단계 둠 |
| 한글 폰트 미적용으로 깨진 모양 | M | M | `custom.css`에서 `font-family`에 한글 폰트(Pretendard 등) 명시, fallback 체인 작성 |
| `baseURL` 슬래시 누락으로 자산 경로 깨짐 | M | M | `https://<username>.github.io/` 끝에 `/` 필수, 워크플로에서 `--baseURL` 재주입 |
| 한글 슬러그가 URL에 그대로 들어가 인코딩 문제 | L | M | front matter에 `slug: "english-kebab"` 명시 권장, 가이드에 포함 |
| 콘텐츠 마이그레이션 중 코드블록·표 깨짐 | M | L | 1차는 텍스트 위주이므로 영향 작음. 가이드에 점검 포인트 명시 |

## Verification Steps
1. **Local build 검증**: `hugo --gc --minify` 0 에러, `public/` 생성, `public/index.html`에 `<title>`이 사용자 사이트 제목으로 채워져 있음
2. **로컬 미리보기**: `hugo server -D`로 http://localhost:1313 확인 — 메인, 글 1개, About 모두 진입 가능
3. **모바일 반응형**: 브라우저 dev tools 375px 너비에서 가로 스크롤 없음, 텍스트 가독
4. **테마 커스터마이징 적용 확인**: dev tools로 `--primary` 값과 폰트가 사용자 지정값과 일치
5. **CI 빌드**: GitHub Actions Run 성공 (녹색 체크), artifact 업로드/배포 모두 통과
6. **Production URL**: `https://<username>.github.io/` 200 OK, 글 1개 진입 가능, 검색·태그·다크모드 토글 동작
7. **First-impression 테스트**: 30초 안에 "이름 + 직무 + 글 목록"이 한 화면에 보이는지 시각 확인

---

## Open Questions for Execution
- GitHub username (Phase A에서 사용자에게 묻기)
- 메인 색상/폰트 (Phase A)
- 첫 게시글 콘텐츠 (Phase A, 없으면 placeholder)
- 사이트 제목/슬로건 (예: "Eunbi's Notes" — Phase A)

## Deferred to v2 (Non-goals reaffirmed)
- 댓글 시스템 (giscus 권장하지만 v2)
- 커스텀 도메인 + DNS
- SEO 고도화 (sitemap.xml은 Hugo 자동 생성, 그 이상은 v2)
- Confluence REST API 자동화 마이그레이션
